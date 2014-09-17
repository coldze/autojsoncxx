autojsoncxx
===========
A header-only library and a code generator to **automagically** translate between **JSON** and **C++** types.

## Overview

JSON is an excellent format for data serialization due to its simplicity, flexibility, portability and human-readable nature. Writing code to parse and generate JSON, however, is not an easy task in a statically typed language. Even with the help of JSON libraries, you need to write a lot of boilerplate code, and convoluted ones if you need to enforce the static typing of C++.

More importantly, maually writing the code means duplication of effort, and duplication is bad for programmers. When your client or PM request a change in feature, many classes (like the class responsible for configuration) will likely change, and you will have to rewrite the code. During the rewrite, time is wasted, people become impatient, and bugs may be introduced when the class definition, parsing and serialization code become out of sync.

*autojsoncxx* is an attempt to solve this problem by automating such process. It is currently still in alpha stage, so expect things to change in the future, or to break down when your class structure is too convoluted.

Dependency: 
* RapidJSON (https://github.com/miloyip/rapidjson) 
* (optional, only the headers) Boost (http://www.boost.org)

## Features

* The parsing/serializing code are *automagically* generated. You don't even need to understand what is proper JSON to use it, although it may help you diagnose problems.
* *Detailed error message*. Not only do you get informed if the JSON is not valid, but you will have a verbose trace back pointing to the location of the problem as well, if the JSON value does not fit your class structure.
* *Ease of use*. Many convience functions are added so that a single function call is enough for most use cases. The library as well as its dependency are header only, so no complicated setup for your build system is needed.
* *Fast*. The underlying JSON engine (RapidJSON) has been benchmarked to be about an order of magnitude faster than other popular JSON libraries. Besides, this library uses its SAX API, obviating the need of constructing a Document Object Model as the intermediate representation. Lastly, the library utilizes C++ templates to generate the algorithm at compile time, so no overhead of runtime indirection (except when error occurs).
* *Flexible framework*. You can add more type support to the library by specializing certain template classes. In addition, whenever a class is generated, you can also parse/serialize an array of such class, a nullable wrapper of such class, another class that contains it, etc.
* *Liberal licence*. Both the library and its dependency are licenced liberally (MIT or BSD-like). Anyone is free to copy, distribute, modify or include in their own projects, be it open source or commercial.

## First example

The code generator reads a JSON file that defines the class structure. An example definition is like this (remember to fully qualify the type name with its namespace)

```javascript
{
    "name": "Person",
    "members":
    [
        ["unsigned long long", "ID", {"required": true}],
        ["std::string", "name", {"default": "anonymous"}],
        ["double", "height"],
        ["double", "weight"],
        ["std::vector<unsigned long long>", "known_associates"]
    ]
}
```

Run the script *autojsoncxx.py* (requires Python 3) on this definition file, and a header file will be generated. It includes a definition for `Person` as well as some helper classes. The `Person` is a `struct` with all members public, meant as a data holder without any additional functionalities.

```bash
python3 autojsoncxx.py --out=person.hpp def.json
```

Remember to add the include directory of *autojsoncxx* and *rapidjson* to your project header search path (no linking is required). 

The below examples uses c++11 features, but the library should also work with c++03 compilers.

### Serialization
```c++
#define AUTOJSONCXX_MODERN_COMPILER 1 // Turn on all the c++11 features of the library
#include <iostream>
#include "person.hpp"

int main()
{
    Person p;
    p.name = "Mike";
    p.ID = 8940220481904ULL;
    p.weight = 70;
    p.height = 1.77;
    p.known_associates = { 149977889346362, 90000134866608, 44412567664 };
    // Use successive push_back() if your compiler is not c++11 ready

    autojsoncxx::to_pretty_json_file("person.json", p);
    return 0;
}
```
This will generate a file `person.json` with contents below:
```javascript
{
    "name": "Mike",
    "ID": 8940220481904,
    "height": 1.77,
    "weight": 70.0,
    "known_associates": [
        149977889346362,
        90000134866608,
        44412567664
    ]
}
```

### Parsing
Now let's try read that back
```c++
#define AUTOJSONCXX_MODERN_COMPILER 1
#include <iostream>
#include "person.hpp"

int main()
{
    autojsoncxx::ParsingResult result;
    Person p;
    if (!autojsoncxx::from_json_file("person.json", p, result)) {
        std::cerr << result << '\n';
        return -1;
    }

    std::cout << "ID: " << p.ID << '\n'
              << "name:  " << p.name << '\n'
              << "height: " << p.height << '\n'
              << "weight: " << p.weight << '\n';

    std::cout << "known associates: ";
    for (auto&& id : p.known_associates)
        std::cout << id << '\t';
    std::cout << '\n';
    return 0;
}
```
### Error handling
If the JSON file is malformed, any decent JSON library will detect it and tell you what goes wrong. But what if the JSON value is perfectly valid, but not layed out the way you expected? Usually you have to manually check the DOM tree against your specification, but this library will automatically generates the necessary code.

Here is valid JSON file

```js
{
    "name": "Mike",
    "ID": 8940220481904,
    "height": 1.77,
    "weight": 70.0,
    "known_associates": [
        "Jack", "Mary"
    ]
}
```

Running through the parsing code, and you will get an error output:

```
Parsing failed at offset 127 with error code 16:
Terminate parsing due to Handler error.

Trace back (last call first):
(*) Type mismatch between expected type "uint64_t" and actual type "string"
(*) Error at array element with index 0
(*) Error at object member with name "known_associates"
```

To programmingly examine the error, you need to query the `autojsoncxx::ParsingResult` class. There are two major groups of error: invalid JSON and mismatch between JSON and C++ class specification. Calls `json_parse_result()` to get a RapidJSON error reporting object, and iterate over the `autojsoncxx::ParsingResult` object for any errors resulting from mapping JSON to C++ types.

```c++
for (auto&& e : result) {
    using namespace autojsoncxx::error;

    switch (e.type()) {
    case UNKNOWN_FIELD: {
        const UnknownFieldError& err = static_cast<const UnknownFieldError&>(e);
        if (err.field_name().find("Version") != std::string::npos)
            std::cerr << "This is a definition of different protocol version\n";
    } break;

    case NUMBER_OUT_OF_RANGE:
        std::cerr << "Maybe you should use a 64-bit integer type instead?\n";
        break;

    case TYPE_MISMATCH: {
        const TypeMismatchError& err = static_cast<const TypeMismatchError&>(e);
        std::cout << "don't you dare use a " << err.actual_type()
                  << " to fool me!\n";
    } break;

    case OBJECT_MEMEMBER: {
        const ObjectMemberError& err = static_cast<const ObjectMemberError&>(e);
        std::cout << "The member " << err.member_name() << " is naughty!\n";
    } break;

    default:
        break;
    }
}
```
## Type support
These types are supported by this library:

* Basic types: `bool`, `int`, `unsigned int`, `long long`, `unsigned long long`, `std::string`
* Array tyeps: `std::vector<>`, `std::deque<>`, `std::array<>`, `std::tuple<>` (this one needs special care)
* Nullable types: `std::shared_ptr<>`
* Object types: any class generated by the script *autojsoncxx.py*.

If you include `<autojsoncxx/boost_types.hpp>`, you will also get support for

* Array types: `boost::container::vector<>`, `boost::container::deque<>`, `boost::array`
* Nullable types: `boost::shared_ptr<>`, `boost::optional<>`

**No raw pointer and reference types are supported**. They do not convey any information about ownership, and will make correct memory management (especially by a code generator) much more difficult.

### Complex types

The supported types can be arbitrarily nested, for example

```c++
#define AUTOJSONCXX_MODERN_COMPILER 1 
#include <iostream>
#include "person.hpp"

int main()
{
    auto test = std::make_tuple(std::vector<std::string>{"A", "BC", "DEF"},
                                9.0, true, std::shared_ptr<int>(),
                                std::make_shared<std::array<Person, 2>>());
    
    std::string str;
    
    // This requires true variadic template support 
    // MSVC 2012 has std::tuple<>, but it is faked with macros
    // Avoid std::tuple<> if your compiler is not strong enough
    autojsoncxx::to_pretty_json_string(str, test);
    
    std::cout << str << '\n';
    return 0;
}
```

Sample output

```js
[
    [
        "A",
        "BC",
        "DEF"
    ],
    9.0,
    true,
    null,
    [
        {
            "ID": 0,
            "name": "anonymous",
            "height": 0.0,
            "weight": 0.0,
            "known_associates": []
        },
        {
            "ID": 0,
            "name": "anonymous",
            "height": 0.0,
            "weight": 0.0,
            "known_associates": []
        }
    ]
]
```
### Self defined type

The core of the library is two template class, `SAXEventHandler` and `Serializer`. The base templates are defined as:

```c++
namespace autojsoncxx {

// The core handlers for parsing
template <class T>
class SAXEventHandler;

// Only the second parameter should be specialized
template <class Writer, class T>
struct Serializer;

}
```

Each of the full or partial specialization of these templates will add new capability to the library. 

Writing the handler is somewhat difficult, so there is some base classes (with no virtual functions) provided. Here is two code snippet about the definition of `std::vector<>` and `std::shared_ptr<>`. You could add your own linear container and nullable wrapper types analogously.

```c++
#if AUTOJSONCXX_HAS_MODERN_TYPES

template <class T>
class SAXEventHandler<std::shared_ptr<T> >
    : public NullableBaseSAXEventHandler<T, SAXEventHandler<std::shared_ptr<T> > > {

public:
    typedef std::shared_ptr<T> smart_pointer_type;

private:
    smart_pointer_type* m_value;

public:
    explicit SAXEventHandler(smart_pointer_type* v)
        : m_value(v)
    {
    }

    bool IsNull() const
    {
        return m_value->get() == 0;
    }

    T* Initialize()
    {
        std::make_shared<T>().swap(*m_value);
        return m_value->get();
    }

    void SetNull()
    {
        m_value->reset();
    }
};

template <class Writer, class T>
struct Serializer<Writer, std::shared_ptr<T> > {
    void operator()(Writer& w, const std::shared_ptr<T>& ptr) const
    {
        if (!ptr)
            w.Null();
        else
            Serializer<Writer, T>()(w, *ptr);
    }
};

namespace utility {
    namespace traits {
    
        // Register with the traits class
        // It is not currently used
        // But essential when I implement support for variant types
        template <class T>
        struct is_simple_type<std::shared_ptr<T> > : public is_simple_type<T> {
        };
    }
}
#endif
```

```c++
namespace autojsoncxx {

// Note the use of *Curiously Recurring Template Pattern*

template <class T, class Allocator>
class SAXEventHandler<std::vector<T, Allocator> >
    : public VectorBaseSAXEventHandler<T, SAXEventHandler<std::vector<T, Allocator> > > {

public:
    typedef std::vector<T, Allocator> vector_type;
    typedef VectorBaseSAXEventHandler<T, SAXEventHandler<std::vector<T, Allocator> > > base_type;

private:
    vector_type* m_value;

public:
    explicit SAXEventHandler(vector_type* v)
        : m_value(v)
    {
    }

    void Push(const T& c)
    {
        m_value->push_back(c);
    }

#if AUTOJSONCXX_HAS_RVALUE

    void Push(T&& c)
    {
        m_value->push_back(AUTOJSONCXX_MOVE(c));
    }

#endif

    bool CheckLength(SizeType)
    {
        // returns false when the container is fixed lengthed
        // and the argument does not match
        return true;
    }

    size_t ExpectedLength() const
    {
        return 0;
    }

    size_t GetCurrentSize() const
    {
        return m_value->size();
    }
};

template <class Writer, class Container, class ValueType, class ConstIteratorType>
struct ContainerSerializer {
    void operator()(Writer& w, const Container& con) const
    {
        w.StartArray();
        for (ConstIteratorType it = con.begin(), end = con.end(); it != end; ++it)
            Serializer<Writer, ValueType>()(w, *it);
        w.EndArray();
    }
};

// Serializers are easy to write, so you can dispense with base classes if you want
template <class Writer, class T, class Allocator>
struct Serializer<Writer, std::vector<T, Allocator> >
    : public ContainerSerializer<Writer, std::vector<T, Allocator>,
                                 typename std::vector<T, Allocator>::value_type,
                                 typename std::vector<T, Allocator>::const_iterator> {
};

}
```

### About tuple types

There are only one tuple type supported `std::tuple` (`boost::tuple` is not supported). Implementing it requires true variadic templates, so for most compilers it is not accessible.

If you want to use it, you need to define both `AUTOJSONCXX_HAS_MODERN_TYPES` and `AUTOJSONCXX_HAS_VARIADIC_TEMPLATE` to be nonzero. The macro `AUTOJSONCXX_MODERN_COMPILER` automatically turns on both two.

The tuple type is mapped to a JSON array of heterogenous types. So `std::tuple<int, std::string, double>` maps to a JSON array of three element of type `Number`, `String`, and `Number` respectively.

During parsing, only the prefix is matched. That is, if the JSON array is longer than the tuple size, the extraneous part will be silently dropped; if the JSON array is shorter than the tuple size, the not-mapped element simply remains untouched.This design is based on the assumption that when you need a heterogeneous array, you probably priotize flexibility over strict conformance.

Note that the matched part still must have compatible type. Support for variant types is planned.

## Other

You can have multiple definition of classes in the same file, simply by making the root an array of definitions.

## To do

* More extensive testing of the library.
* Testing it on compilers other than clang.
* Automatic detection of compiler support for c++11 features.
* More documentation and comments.
* Map and variant types support.
* The current output of code generator is not formatted uniformly. You can simply run a code formatter such as `astyle`, `clang-format` or `uncrustify` on it to make it human readable. In the future, options will be provided to automatically call one of the formatters.
* Ability to separate the class definition and its helper classes in the output. Useful if you want to avoid the clutter when looking at what members your class contains.