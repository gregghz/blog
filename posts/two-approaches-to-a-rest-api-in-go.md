---
title: 'Two Approaches to a REST API in Go'
date: '2013-05-22'
description:
tags: [golang, reflection, REST]
---

Below I will outline two approaches to writing a flexible REST API in Go. The first approach will use Go's reflect package to allow for the dynamic creation of URLs based on which methods are present. It will dynamically lookup method names when a request comes in and it will simply return a 404 error to a user if they make a request to an unknown method. The second approach will will sacrifice a bit of the dynamic nature of the first approach (you have to explicitly register URLs) but it will typically be much faster and I feel that the code is much more clear.

Approach One
============

This approach uses Go's reflect package to allow for a flexible API
for adding new endpoints. We'll simply implement an
[http.Handler](http://golang.org/pkg/net/http/#Handler) and use the
method's attached to the handler to control the program
logic. Requests are automatically mapped to methods defined on the
Handler type. For example, given the code below, a POST request to
/pagetwo will generate the response "POST Page Two!" while a GET
request to the same url will generate "GET Page Two!" The main
weakness of this approach is that in general, reflection is slow.

    package main
    
    import (
        "fmt"
        "log"
        "net/http"
        "reflect"
        "strings"
    )
    
    type Handler struct {}
    
    func (f *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        // A special case for the index
        path := r.URL.Path
        if path == "/" {
            path = "/index"
        }

        // we need to do some processing to turn the url into a callable name
        // after a POST request to /user/login, call_name will be "User_login_POST"    
        call_name := strings.ToUpper(path[1:2]) // capitalize the first letter .. we can only call exported methods
        call_name += strings.Replace(strings.ToLower(path[2:]), "/", "_", -1)
        call_name += "_" + r.Method
        
        method := reflect.ValueOf(f).MethodByName(call_name)
        if method.IsValid() {
            method.Call([]reflect.Value{reflect.ValueOf(w), reflect.ValueOf(r)})
        } else {
            // send off a 404 error here.
            w.WriteHeader(http.StatusNotFound)
        }
    }
    
    func (h *Handler) Index_GET(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Index")
    }
    
    func (h *Handler) Pagetwo_GET(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "GET Page Two!")
    }
    
    func (h *Handler) Pagetwo_POST(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "POST Page Two!")
    }
    
    func main() {
        s := &http.Server{
            Addr: ":2056",
            Handler: new(Handler),
        }
        log.Fatal(s.ListenAndServe())
    }


Approach Two
============

I prefer this approach because it's more explicit about what is
happening and doesn't depend on any knowledge of the implementation of
ServeHTTP. It also allows you to have handlers that are a part of any
package. Perhaps even more important than all that is that this
version is easier to understand without the need for extraneous
comments. Just add handlers with AddHandler. ServeHttp does a lookup
in handlers for each request made. If it's found, it executes that
handler. If none is found, it returns a 404 error. I should note that
this could be cleaned up quite a bit (the handler global should be
eliminated, for example), but I think this is a good starting point.

    package main
    
    import (
        "fmt"
	    "log"
	    "net/http"
    )
    
    var handlers = map[string]http.Handler{}
    
    type Handler struct {
    }
    
    func (f *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    path := r.URL.Path
    
    if handler, ok := handlers[path+"_"+r.Method]; ok {
        handler.ServeHTTP(w, r)
        } else {
            w.WriteHeader(http.StatusNotFound)
        }
    }
    
    func AddHandler(path, method string, h func(http.ResponseWriter, *http.Request)) {
        handlers[path+"_"+method] = http.HandlerFunc(h)
    }
    
    func index_GET(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Index")
    }
    
    func main() {
        AddHandler("/", "GET", index_GET)
        AddHandler("/pagetwo", "GET", func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprintf(w, "GET Page Two!")
        })
        AddHandler("/pagetwo", "POST", func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprintf(w, "POST Page Two!")
        })
        
        s := &http.Server{
            Addr:    ":2056",
            Handler: new(Handler),
        }
        log.Fatal(s.ListenAndServe())
    }
