# Implement XML and HTML parsing with streaming SAX-style and DOM approaches

**Category:** Serialization and Data Formats  
**Standard:** C++17  
**Reference:** <https://pugixml.org/> · <https://rapidxml.sourceforge.net/>  

---

## Topic Overview

XML remains common in enterprise systems, configuration files, and document formats. There are two fundamentally different ways to parse it, and choosing the wrong one can either leave you fighting a terrible API or exhausting all available memory.

**DOM** (Document Object Model) loads the entire XML tree into memory as a navigable tree of nodes. **SAX/streaming** fires callbacks as the parser encounters each element, keeping memory usage proportional to the nesting depth rather than the document size. Both have their place.

### DOM Parsing with pugixml

DOM is the right choice when you need to navigate the tree in arbitrary order - jumping from a leaf back to its parent, applying XPath queries, or building an in-memory representation. pugixml makes this very comfortable. Load the document once, then traverse it as many times as you like.

```cpp
#include <pugixml.hpp>
#include <iostream>
#include <string>

void dom_example() {
    pugi::xml_document doc;
    auto result = doc.load_string(R"(
        <library>
            <book id="1">
                <title>Effective Modern C++</title>
                <author>Scott Meyers</author>
                <year>2014</year>
            </book>
            <book id="2">
                <title>C++ Concurrency in Action</title>
                <author>Anthony Williams</author>
                <year>2019</year>
            </book>
        </library>
    )");

    if (!result) {
        std::cerr << "Parse error: " << result.description() << "\n";
        return;
    }

    // Navigate the DOM tree
    for (auto book : doc.child("library").children("book")) {
        std::cout << "Book #" << book.attribute("id").as_int() << ": "
                  << book.child("title").text().get() << " by "
                  << book.child("author").text().get() << " ("
                  << book.child("year").text().as_int() << ")\n";
    }

    // XPath queries
    auto nodes = doc.select_nodes("//book[year>2015]");
    std::cout << "\nBooks after 2015:\n";
    for (auto& node : nodes) {
        std::cout << "  " << node.node().child("title").text().get() << "\n";
    }
}
```

Notice the XPath query `"//book[year>2015]"` - that is the kind of expressive filtering that only works when you have the whole tree loaded. With a SAX parser you would have to implement that filtering logic yourself with a state machine.

### SAX-Style Streaming with Expat Wrapper

SAX (Simple API for XML) is event-driven. You provide a handler object with callbacks, and the parser calls them as it scans through the document. Your handler accumulates whatever state it needs - there is no tree to query after the fact. The memory footprint is tiny: only the current element stack needs to be in memory at any point.

```cpp
#include <iostream>
#include <string>
#include <functional>
#include <stack>

// Conceptual SAX callback interface
struct SAXHandler {
    virtual void on_start_element(const std::string& name,
                                   const std::vector<std::pair<std::string, std::string>>& attrs) = 0;
    virtual void on_end_element(const std::string& name) = 0;
    virtual void on_text(const std::string& text) = 0;
    virtual ~SAXHandler() = default;
};

class BookCounter : public SAXHandler {
    int count_ = 0;
    bool in_title_ = false;
public:
    void on_start_element(const std::string& name,
                           const std::vector<std::pair<std::string, std::string>>&) override {
        if (name == "book") ++count_;
        in_title_ = (name == "title");
    }

    void on_end_element(const std::string&) override {
        in_title_ = false;
    }

    void on_text(const std::string& text) override {
        if (in_title_) {
            std::cout << "  Title: " << text << "\n";
        }
    }

    int count() const { return count_; }
};

// Usage: feed XML to handler chunk by chunk
// Memory usage: O(depth) vs DOM's O(document_size)
```

The `in_title_` flag is a typical SAX pattern: you track what element you are currently inside so that `on_text()` knows how to interpret the text it receives. The more complex your extraction logic, the more state flags you end up managing - which is why SAX feels harder to use than DOM even though the callback interface itself is simple.

### DOM vs SAX Decision Matrix

If you are unsure which approach to reach for, this table gives you the practical answer for most scenarios.

| Factor | DOM (pugixml) | SAX/Streaming |
| --- | --- | --- |
| Memory | O(document size) | O(tree depth) |
| Random access | Yes | No (forward only) |
| Ease of use | Very easy | More complex |
| Speed | Fast for small docs | Better for huge docs |
| Use case | Config files, small data | Log processing, >100MB XML |

---

## Self-Assessment

### Q1: Write XML from a C++ struct using pugixml

Writing XML with pugixml uses the same tree API as reading - you build nodes programmatically and then serialize with `doc.save()`. The `std::ostringstream` sink gives you the XML as a `std::string`, which is convenient for returning from a function.

```cpp
#include <pugixml.hpp>
#include <sstream>
#include <string>

struct Config {
    std::string host;
    int port;
    bool debug;
};

std::string to_xml(const Config& cfg) {
    pugi::xml_document doc;
    auto root = doc.append_child("config");
    root.append_child("host").text().set(cfg.host.c_str());
    root.append_child("port").text().set(cfg.port);
    root.append_child("debug").text().set(cfg.debug);

    std::ostringstream oss;
    doc.save(oss, "  ");
    return oss.str();
}
// Produces:
// <?xml version="1.0"?>
// <config>
//   <host>localhost</host>
//   <port>8080</port>
//   <debug>true</debug>
// </config>
```

### Q2: Why is SAX parsing preferred for very large XML files

DOM parsing loads the entire XML tree into memory. A 1 GB XML file typically requires 2-5 GB of RAM for the DOM representation, because each node is a heap-allocated object with pointers. SAX/streaming parsers process one element at a time, using O(depth) memory regardless of file size. For log files, data feeds, and scientific data in XML, streaming is the only practical approach - building a DOM from a 1 GB file may simply exhaust available memory before you can read a single value.

### Q3: Handle XML namespaces correctly

XML namespaces are one of the places where "simple" XML turns out to be less simple. The key thing to know about pugixml is that it does not resolve namespaces automatically - the prefix is just part of the element name as far as the library is concerned. If the XML uses `app:config`, you query for `"app:config"`, not just `"config"`.

```cpp
// XML with namespaces:
// <root xmlns:app="http://example.com/app">
//   <app:config>
//     <app:port>8080</app:port>
//   </app:config>
// </root>

// pugixml does NOT process namespaces automatically.
// Element names include the prefix: "app:config"
// Use the full prefixed name in queries:
auto node = doc.child("root").child("app:config").child("app:port");
```

If you need full namespace URI resolution (i.e., treating `app:port` and `x:port` as the same element when both `app` and `x` map to the same URI), you will need a library that handles namespaces, or you will need to do the URI mapping yourself.

---

## Notes

- **pugixml** is the recommended C++ XML library: fast, optionally header-only, with XPath support.
- **RapidXML** is faster for parsing but lacks XPath and has no output support.
- For HTML parsing (which is often malformed XML), use **Gumbo** or **lexbor**.
- Avoid `tinyxml2` for new projects - pugixml is faster and has a better API.
- XML is verbose; prefer JSON or binary formats for new systems unless XML is mandated.
