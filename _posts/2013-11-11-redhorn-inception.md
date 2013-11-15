---
layout: post
title:  "RedHorn: inception"
date:   2013-11-11 10:57:03
---
## Foreword

Probably every developer who had to implemenent yet another middleware for a new project had thoughts about
flexible and more or less generic solution. Which would save save them from that tedious and monotonous work
and their time spent describing each packet, writing serialization and deserialization code etc. Also if your
project requires heterogenous communication in terms of techology and/or progamming language which is used to
implement diffent components of system, we've got one more problem with syncronization of implementations of
transport protocol. For example, to establish communication between server written in C++ and simple client
script written in Python, we need all packet involved in communition to be accesible from python and c++ code.
Which leads us to double descrition of protocol, custom build steps, or even worse use of json or xml insted
of binary protocol which could cause significant performance degradation, especially for hpc grid. Sure there
are a lot of approaches to deal with this problem, all of them has their benefits and drawbacks. Following is
just yet another approach, some techincs and ideas, most of them inspired by Andrei Alexandrescu's Modern C++
Design.

## C++ structure with field you can iterate by

Haters are gonna say that you just would have such problem if you used one of "modern" programming language
with reflection such as C#. Lets leave his holly war for another blog post. But still, what would you say if
you could get c++ structure with field you can iterate by, practically without overhead and in compile time?
Yeah, I know, that sounds like poorly-understandable and "powerful" feature which involves template magic.
If you think so, you are right, but anyway that could be fun.

So what do we need for it? First of all we need some compile-time meta information which contains field type,
filed size and offset from the beginning of structure. For that purpose Alexandrescu's type-lists suites best.
If you are no familiar with TypeLists, probably it's a good time to fill those gaps and read about Loki or
similiar libraries. But actully we are going to use variadic templates insted of type-lists, which is pretty
the same but with less amount of code. if you are not familiar with variadic templates, you know what to read,
I mean C++11 standart or at least wikipedia's article about C++11.

Lets say we have list of instantiations of following template:

{% highlight c++ %}
template <int _offset, int _size, typename _info>
struct field
{
    static const int offset = _offset;
    static const int size = _size;
    typedef _info info;
};
{% endhighlight %}

where info is just some addition information about field such as string literal with name of field,
typedef of field actual type etc.

{% highlight c++ %}
struct field_info {
    typedef int type;
    static const char* name;
};
const char* field_info::name = "foo";
{% endhighlight %}

That would be probably enough for our purposes, but how to get it with minimum amount of code and define
directives? It's not that hard when you find out at least how to count number of macros invocation used
to describe structure in compile-time.

## How to count number of macros invocation

The problem is that we cannot simply redefine type or const variable in c++

{% highlight c++ %}
typedef empty_list list;
typedef type_list<int, list> list;
typedef type_list<float, list> list;

const int foo = 5;
const int foo = foo + 1;
{% endhighlight %}

That trick isn't gonna work with c++.

But here what is gonna work:

{% highlight c++ %}
//initialization
#define INIT(SMALLEST_NUMBER)                               \
    template<int _idx>                                      \
    struct last_one                                         \
    {                                                       \
        static const int count = last_one<_idx - 1>::count; \
        typedef typename last_one<_idx - 1>::list list;     \
    };                                                      \
                                                            \
    template<>                                              \
    struct last_one<0>                                      \
    {                                                       \
        static const int count = SMALLEST_NUMBER;           \
        typedef empty_list list;                            \
    }

//iteration
#define ITERATION(CONSTANTLY_INCREACING_NUMBER, TYPE)                   \
    template<>                                                          \
    struct last_one<CONSTANTLY_INCREACING_NUMBER>                       \
    {                                                                   \
        static const int count = last_one<CONSTANTLY_INCREACING_NUMBER - 1>::count; \
        typedef typename type_list<TYPE, last_one<CONSTANTLY_INCREACING_NUMBER - 1> >::list list; \
    }

//getting results
#define RESULTS(CONSTANTLY_INCREACING_NUMBER)                           \
    typedef typename type_list<int, last_one<CONSTANTLY_INCREACING_NUMBER - 1> >::list list; \
    const int count = last_one<CONSTANTLY_INCREACING_NUMBER - 1>::count

INIT(0);
ITERATION(2, int);
ITERATION(4, int);
ITERATION(5, float);
RESULTS(10);
{% endhighlight %}

That's good but not good enough. You have to providet that constantly increacing numbers and it annoys.
But here clever solution comes to rescue, __LINE__ macro TA-DA! Really with minor limitation you can use
it and be pretty sure that all thouse numbers are constantly increacing.


## Back to our sheeps

Now when we've figured out how to count and account out fields we can get what we wanted.

{% highlight c++ %}
#ifndef PACKET__PACKET_H
#define PACKET__PACKET_H

#include <cstddef>
#include <memory>

#include "../type_list.h"

namespace redhorn
{
    template <typename _type>
    std::string str(const _type &value);

    namespace _packet
    {
        template <int _offset, int _size, typename _info>
        struct field
        {
            static const int offset = _offset;
            static const int size = _size;
            typedef _info info;
        };
    
        template <typename _first, typename _second>
        struct type_pair 
        {
            typedef _first first;
            typedef _second second;
        };
    
        template <typename _type_pair>
        struct packet : public _type_pair::first
        {
            typedef typename _type_pair::second fields;
            std::string str() const 
            {
                return redhorn::str<packet<_type_pair>>(*this);
            }
        };
    };
    using _packet::packet;
};
#define PACKET(_struct_name)                                    \
    namespace NS##_struct_name {                                \
    template <int index>                                        \
    struct last_one                                             \
    {                                                           \
        typedef typename last_one<index - 1>::type type;        \
        typedef typename last_one<index - 1>::fields fields;    \
    };                                                          \
    template <>                                                 \
    struct last_one<__LINE__>                                   \
    {                                                           \
        typedef redhorn::type_list::empty fields;               \
        struct type                                             \
        {                                                       \
            static const char* _name;                           \
        };                                                      \
    };                                                          \
    const char* last_one<__LINE__>::type::_name = #_struct_name 



#define FIELD(_type, _name)                                             \
    struct info_##_name {                                               \
        typedef _type type;                                             \
        static const char* name;                                        \
    };                                                                  \
    const char * info_##_name::name = #_name;                           \
    struct derived_##_name : public last_one<__LINE__ - 1>::type        \
    {                                                                   \
        _type _name;                                                    \
    };                                                                  \
    template <>                                                         \
    struct last_one<__LINE__>                                           \
    {                                                                   \
        typedef last_one<__LINE__ - 1> previous;                        \
        typedef derived_##_name type;                                   \
        typedef typename  redhorn::type_list::concat< redhorn::_packet::field<offsetof(derived_##_name, _name), \
                                                                              sizeof(_type), \
                                                                              info_##_name>, \
                                                      typename previous::fields>::type fields; \
    }

#define TEKCAP(_struct_name)                                            \
    };                                                                  \
    typedef redhorn::packet< redhorn::_packet::type_pair<typename NS##_struct_name::last_one<__LINE__>::type, \
                                                         typename NS##_struct_name::last_one<__LINE__>::fields> >  _struct_name


namespace redhorn 
{
    namespace _packet
    {
        PACKET(empty_packet);
        TEKCAP(empty_packet);
    };
    using _packet::empty_packet;
};

#endif // PACKET__PACKET_H
{% endhighlight %}

Cool, huh?

## Now what?

And now when we have implemented this "powerful" and poorly-understandable feature it's time to think
what to do with it? Well, that's obvious, now we can write compile-time algorithms which create string
literal with packet's description, serialize/deserialize packet, describe packet in python style and
more. Also it's worth to note that all those algorithm will be optimized and inlined by compiler so
actually we'll get linear code without loops and branching which means best perfomance we could get
(And yes, we all know that anyway bottleneck will be in actual writing into socket, but it’s still
nice to know that you’ve done your best)

## To be continue

Furner we're gonna talk about actual implementation of algorithms mentioned above and some other
iteresting stuffs.

