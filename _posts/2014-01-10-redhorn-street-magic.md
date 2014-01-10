---
layout: post
title:  "RedHorn: A little bit of street magic (Draft)" 
date:   2014-01-10 23:00:03
---

## And hi again

Hi, my name is Johnny Knoxville, and today we will try to write piece of actually usefull code.
So let's start with iteration over fields of our brand new packets.

{% highlight c++ %}
template <typename... _list>
struct print_field_name;

template <typename _head, typename... _tail>
struct print_field_name<_head, _tail...>
{
    static void print()
	{
		print_filed_name<_tail...>::print();
		puts(_head::info::name);
	}
};

template<>
struct print_field_name<>
{
    static void print()
	{
	}
};
{% endhighlight %}

And what we've got here? This is simpliest but still quite usefull (at least for debug purpose) code
which should print out list names of fields. You may wonder why do we use template structs with
static method insted of simple template function. Well we do that because it's easier, you don't 
have to worry about interaction  between function overloading and template function specialization.
You may not agree with such approach, but you should admit that template structure specialization
mechanism is much more straightforward and easier to understand. Moreover we always can write
simple template function as user friendly interface.

{% highlight c++ %}
template <typename _packet>
void print_fields()
{
	print_field_name<typename _packet::fields>::print();
}
{% endhighlight %}

##One step forward

Writing "printer" for field names, was easy. But when you need to access to values stored in packet,
things become a bit more complicated. Here we will need field's offset and it's actual type.

{% highlight c++ %}
template <typename... _list>
struct print_field_name;

template <typename _head, typename... _tail>
struct print_field_name<_head, _tail...>
{
	template <typename _type>
    static void print(const _type &value)
	{
		print_filed_name<_tail...>::print(value);
		std::cout<<_head::info::name<<" : "
				 <<(*(typename _head::info::type*)((char*)(&value) + _head::offset))
				 <<std::endl;
	}
};

template<>
struct print_field_name<>
{
	template <typename _type>
    static void print(const _type &value)
	{
	}
};
{% endhighlight %}

Yeah, this type casts is a dirty trick, but there is nothing we can do. All we can
do is accept it as inevitable evil and forgive.

##One more thing

We need one more specialization for template print_field_name, if we want pass packet::fields
as template parameter. That's because we packet::fields is instantiation of type_list::list,
not template parameter pack. Thus something like packet::fields... won't work.

{% highlight c++ %}
template<_list...>
struct print_field_name<type_list::list<_list...> >
{
	template <typename _type>
    static void print(const _type &value)
	{
		print_field_name<_list...>::print(value);
	}
};
{% endhighlight %}

So now, now our public interface can be implemented like this:

{% highlight c++ %}
template <typename _packet>
void print_fields(const _packet &packet)
{
	print_field_name<typename _packet::fields>::print(packet);
}

{% endhighlight %}

##Getting serious

Structure with "auto-generated" serialization/deserialization, isn't it what we were are here for?
Considering last listing it seems really easy to implement desirable. Yet hold you horse, cowboy.
That would be unforgivable wastefull to issue syscall for each field in structure. We'd better 
use some buffer to accumulate data or io vector to perform several writes/reads with one syscall.
In our case we chose io vector, but before we start implementing we should consider serialization of
variable lenght (strings or arrays) or nullable (smart pointers) fields. Natural approach for serialization
of such fields, is to include
addition flag (in case of nullable field) or integral value to repersent lenght of field (in case of valiable
lenght field). For us that means two thing:

1. (not important at this point) While reading structure, right before reading variable lenght 
or nullable field we have to issue syscall and actully read all data followed by current filed,
because this last readed value can affects next elements in io vector.

2. (important one) It may be necessary to create some temporary variable (e.g. int variable which stores
lenght of string) and put its address into io vector. All those variables should be destroyed 
when write syscall issued and io vector cleared. Thus we need some kind vector which stores pointers
to such temporary variables.

Considering all written above, we get following code:

builtin/binary.h
{% highlight c++ %}
#ifndef BUILTIN_BINARY_H
#define BUILTIN_BINARY_H

#include <vector>

#include <sys/uio.h>

#include "../utils.h"

namespace redhorn
{
    namespace _binary
    {
        template <typename _type>
        struct serialize_iovec
        {
            static void fill_write(std::vector<iovec> &write_vec, _type *value, std::vector<void*> &cleanup)
            {
                iovec tmp = {value, sizeof(_type)};
                write_vec.push_back(tmp);
            }

            static void fill_read(int fd, std::vector<iovec> &read_vec, _type *value)
            {
                iovec tmp = {value, sizeof(_type)};
                read_vec.push_back(tmp);
            }
        };
    };
    
    namespace binary
    {
        template <typename _type>
        static void write(int fd, _type &value)
        {
            std::vector<iovec> write_vec;
            std::vector<void*> cleanup;
            try 
            {
                _binary::serialize_iovec<_type>::fill_write(write_vec, &value, cleanup);
                utils::safe_writev(fd, write_vec);
            }
            catch (...)
            {
                for (size_t i = 0; i < cleanup.size(); ++i)
                    free(cleanup[i]);
                throw;
            }
            for (size_t i = 0; i < cleanup.size(); ++i)
                free(cleanup[i]);
                
        };
            
        template <typename _type>
        static void read(int fd, _type &value)
        {
            std::vector<iovec> read_vec;
            _binary::serialize_iovec<_type>::fill_read(fd, read_vec, &value);
            if (read_vec.size() > 0)
                utils::safe_readv(fd, read_vec);
        };
    };
};

#endif // BUILTIN_BINARY_H
{% endhighlight %}


packet/binary.h
{% highlight c++ %}
#ifndef PACKET_BINARY_H
#define PACKET_BINARY_H

#include "packet.h"

#include "../builtin/binary.h"

namespace redhorn
{
    namespace _binary
    {
        template <typename... _list>
        struct serialize_fields;
        
        template <typename _head, typename... _tail>
        struct serialize_fields<_head, _tail...>
        {
            template <typename _type>
            static void fill_write(std::vector<iovec> &write_vec, _type *value, std::vector<void*> &cleanup)
            {
                serialize_fields<_tail...>::fill_write(write_vec, value, cleanup);
                serialize_iovec<typename _head::info::type>::fill_write(write_vec, (typename _head::info::type*)((char*)value + _head::offset), cleanup);
            }

            template <typename _type>
            static void fill_read(int fd, std::vector<iovec> &read_vec, _type *value)
            {
                serialize_fields<_tail...>::fill_read(fd, read_vec, value);
                serialize_iovec<typename _head::info::type>::fill_read(fd, read_vec, (typename _head::info::type*)((char*)value + _head::offset));
            }
        };

        template <>
        struct serialize_fields<>
        {
            template <typename _type>
            static void fill_write(std::vector<iovec> &write_vec, _type *value, std::vector<void*> &cleanup)
            {
            }

            template <typename _type>
            static void fill_read(int fd, std::vector<iovec> &read_vec, _type *value)
            {
            }
        };

        template <typename... _fields>
        struct serialize_fields<type_list::list<_fields...> >
        {
            template <typename _type>
            static void fill_write(std::vector<iovec> &write_vec, _type *value, std::vector<void*> &cleanup)
            {
                serialize_fields<_fields...>::fill_write(write_vec, value, cleanup);
            }
            template <typename _type>
            static void fill_read(int fd, std::vector<iovec> &read_vec, _type *value)
            {
                serialize_fields<_fields...>::fill_read(fd, read_vec, value);
            }

        };


        template <typename _t>
        struct serialize_iovec< packet<_t> >
        {
        private:
            typedef  packet<_t> _type;
        public:
            static void fill_write(std::vector<iovec> &write_vec, _type *value, std::vector<void*> &cleanup)
            {
                serialize_fields<typename _type::fields>::fill_write(write_vec, value, cleanup);
            }
            
            static void fill_read(int fd, std::vector<iovec> &read_vec, _type *value)
            {
                serialize_fields<typename _type::fields>::fill_read(fd, read_vec, value);
            }
        };
    };
};

#endif // PACKET_BINARY_H
{% endhighlight %}

##Next friday

Next time we'll discuss in details serialiation of some variable lenght fields, and some approaches
to organize your template library in extensible and flexible way.
See ya, bye.
