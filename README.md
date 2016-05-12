# Writing node.js Bindings - General Principles

This material is about creating node.js interfaces for native libraries written in C. It is not so much about the mechanics of writing the code, or about the structure of a npm package containing a native addon, but about the various situations you are likely to face when writing addons. It starts with the assumption that it is best to create an addon that provides as little abstraction as possible so as to allow you to provide a Javascript API to the consumers of your project that is itself written in Javascript. The portion of the node.js documentation that describes native addons, the V8 reference, and the reference provided by the Native Abstractions for Node project give you an ample toolset for creating native addons. Once you've managed to create your first native addon, and are ready to bring a complete native library into the node.js world, you will be faced with having to translate the artifacts of the C language into Javascript concepts.  At that point, it pays to be systematic. It is beneficial to break the library's API into constants, enums, structures, and functions, and bind each onto the node.js module you are creating in as automated a fashion as possible. In my presentation, I will describe some patterns for dealing with data structures, pointers, and callbacks one needs to pass to the native library. Finally, I will show some examples of projects where I've applied these concepts.

# Assumption

It is best to write as little logic as possible into the bindings itself. Ideally the addon is a one-to-one mapping of the underlying native API itself. Convenient abstractions can then be implemented in Javascript.

Of course the underlying language has artifacts that need not be mapped.

  0. For example, C has ```void *user_data``` with its function pointers, but that's only to provide room for context. Javascript is the king of context, so you don't need to map this mechanism into Javascript. OTOH, you need this mechanism for the <a href="#callbacks">internals of the bindings</a>.
  0. Another example is pointers that are filled out by the native side. In that case you can have the binding accept an empty object the properties of which it fills out (a receptacle) or you can return a new object you create inside the binding. But what if the native function additionally returns a result code (like ```errno``` for example)? Do you retain one-to-one-ness and use a receptacle object or do you break one-to-one-ness and return an object upon success and an error code otherwise? Do you throw the error code as an exception (again, breaking one-to-one-ness)? Your call.

# General principles

  0. Trust data from the native side, but validate data from the JS side. Do not directly cast a JS value to the type needed by the native API. Instead check that it contains the type needed by the native API first. If it does, cast it and proceed. If it doesn't, throw a TypeError with text naming the parameter that failed validation and return immediately.
  0. Avoid argument juggling. IOW, enforce that there be a certain number of arguments and that each argument have a certain type. At most, allow an argument to optionally be ```null``` (especially for strings).
  0. Make the portion of the stack that starts with the binding mimic the JS behaviour of throwing an exception when needed and tearing down afterwards. This means that the binding only sets the return value if no exceptions were thrown, otherwise it returns early. This in turn means that functions called by the binding (except for the underlying native API, the signature of which we cannot influence) have to return a boolean indicating whether the binding is to immediately return. Internally, these helper functions perform the setup required for later calling the native API and, if validation or other errors occur in the process, they throw an exception and ```return false;```. The fact that the binding checks the return value only for the purposes of deciding whether to return immediately rather than using the return value to decide whether to throw an exception and then return is a design choice. The idea is to be similar to JS in that we throw the exception nearest to the point of failure, rather than returning ```false``` and expecting the caller to throw the exception.
    ```C++
    // Add examples for each case, highlighting the tradeoff between error message specificity (what failed) and error location specificity (what were you trying to do when it failed)
    ```

# Situations

## The easy case

This is a function that accepts primitive values and returns a primitive value:
```C
double area_of_oval(double x_radius, double y_radius);
```
In this case the binding is easy. It accepts a list of arguments such that:
  0. The list is two arguments long
  0. The type of each argument is a JS number that can be cast as a double.
 
Exceptions are only thrown if it turns out that the arguments do not contain values that can be cast as doubles. The return value is constructed from the return value of the native API.

## Data
This is any kind of data that does not fit into a primitive value that the language can handle directly. The distinction from primitive types is that now we're dealing with pointers. There are many ways in which data can be dealt with, so let's break up the use cases and look at each in turn. There are two orthogonal ways in which we can break down how we deal with data: On the one hand there's the case of the pointer the native API eventually expects back via a later call vs. the pointer it gives us for our information. On the other hand there's the pointer to a structure we can copy entirely vs. the "magic" pointer to a structure that is potentially even opaque to us - a handle.

### The simple case - "here's the data you wanted"
This is an API of the form:
```C
struct coordinates {
  double latitude;
  double longitude;
  double altitude;
};
/* Both APIs return non-NULL on success */
const struct coordinates *find_restaurant_by_name(const char *name);
const char *find_restaurant_at_coordinates(const struct coordinates *location);
```
The structure of type ```coordinates``` can be converted to/from Javascript without allocating any memory and worrying about who owns the pointers. In the binding for ```find_restaurant_by_name()``` we construct a JS object from the pointer returned by the API and set it as the return value or, if the API returns ```NULL```, we set the return value to a JS ```null```.

In the case of the second API, the binding accepts a Javascript object which has properties named after the members of the ```cordinates``` structure. We declare a local C structure, retrieve each property of the object passed in, assert that it is of the correct type, throwing an exception and returning immediately if it isn't, and casting the value as a double, while assigning it to the corresponding C structure member.

In general, it's a good idea to create a pair of helper functions that will convert the C structure to a JS structure and back. In [NAN](https://github.com/nodejs/nan/) parlance, this is what that looks like:
```C++
bool c_coordinates(v8::Local<v8::Object> jsCoordinates, struct coordinates *location) {

  // Define a local structure
  struct coordinates localLocation = { 0.0, 0.0, 0.0 };

  // latitude
  Local<Value> jsLatitude = Nan::Get(jsCoordinates,
    Nan::New("latitude").ToLocalChecked()).ToLocalChecked()
  if (!jsLatitude->IsFloat()) {
    Nan::ThrowTypeError("Coordinates latitude must satisfy IsFloat");
    return false;
  }
  localLocation.latitude = Nan::To<double>(jsLatitude).FromJust();

  // longitude
  ...

  // Don't touch the passed-in location until we know for sure that we
  // have all the information for successfully creating the structure.
  *location = localLocation;
  return true;
};

Local<Object> js_coordinates(const struct coordinates *location) {
  Local<Object> returnValue = Nan::New<Object>();
  
  returnValue->Set(Nan::New("latitude").ToLocalChecked(),
    Nan::New(location->latitude));
  returnValue->Set(Nan::New("longitude").ToLocalChecked(),
    Nan::New(location->longitude));
  returnValue->Set(Nan::New("altitude").ToLocalChecked(),
    Nan::New(location->altitude));

  return returnValue;
}
```
You can maximize code reuse and increase the readability of the ```c_<structurename>()``` function if you create a macro for performing the validation. For example,
```C++
#define VALIDATE_VALUE_TYPE(value, typecheck, message, failReturn) \
    if (!(value)->typecheck()) { \
        Nan::ThrowTypeError(message " must satisfy " #typecheck "()"); \
        return failReturn; \
    }
```

### The complex case - handles

In this case the underlying API gives you a "magic" pointer that identifies a resource the API maintains internally. The actual structure may or may not be opaque, but the value of the pointer is important. Note that by its very nature, such an API has to provide at least two functions: one for obtaining the "magic" pointer, and one for informing the API that it is no longer needed and may be disposed of. Here's an example API:

```C
typedef void *RemoteResourceHandle;

RemoteResourceHandle remote_resource_retrieve(const char *host, uint16_t port);
bool remote_resource_set_int(RemoteResourceHandle resource, const char *key, int value);
bool remote_resource_release(RemoteResourceHandle c_handle);
```

How do we bind this API? We could do something simple and treat the handle as an array of bytes. We don't even need to know whether pointers are 32 bit values or 64 bit values. We simply create an array of length ```sizeof(RemoteResourceHandle)``` and store the pointer as an array of bytes. We can even create functions

```C++
Local<Array> js_RemoteResourceHandle(RemoteResourceHandle handle);
bool c_RemoteResourceHandle(Local<Array> jsHandle, RemoteResourceHandle *c_handle);
```

and basically pass the pointer blindly into Javascript land. This is not a very good solution since we have chosen not to trust anything coming from the Javascript side, and we have no way of verifying that the integrity of the pointer value is preserved when we get it back in the binding for ```remote_resource_set_int()``` and ```remote_resource_release()```. It would be really nice if we could create a Javascript object and assign a property to it which is not visible in Javascript land. Fortunately, Javascript allows us to create new types of objects, and V8 exposes this functionality to the native side as well, with the important native-only capability of assigning a number of arbitrary pointers to an object instance.

Before diving into the process of passing a handle into Javascript we should consider an important aspect of handles. Whenever we store the value of the handle, we risk creating a copy that will become stale. That is, for a given handle, we risk passing it to ```remote_resource_set_int()``` after having already called ```remote_resource_release()```. This is especially true in Javascript code, which can be highly asynchronous, in turn making the tracking of the sequence in which the above APIs get called a fairly difficult task. Fortunately, Javascript itself provides us with an excellent template for dealing with handles. Consider ```setTimeout()```:

  0. 
  
  ```JS
  var x = setTimeout( function() {}, 1000 );
  
  // What exactly does x now hold?
  ```
  The function returns a "magic" value, which is completely useless for anything other than passing it to ```clearTimeout()```. Not even the type of the value is certain. In browsers, it's usually an integer, however, in node.js it's an object. All we know is that it can be stored in a variable, and can be copied from variable to variable.
  0. 
  
  ```JS
  clearTimeout( x );
  ```
  When we pass this "magic" value to ```clearTimeout()``` the only circumstance under which anything happens is if the timeout is still active, in which case the effect of ```clearTimeout()``` will be that the callback it was meant to run in a delayed fashion will never be run. In all other circumstances, passing such a value to ```clearTimeout()``` will have no effect.
  0. 
  
  ```JS
  var x = setTimeout( function() {}, 1000 );
  x = null;
  ```
  If we overwrite all references to the "magic" value, we have no way of removing the timeout. This is not a big poblem in the case of a timeout, because it will simply run once and then never again. However, in the case of ```setInterval()```, overwriting the return value of the function means that the interval has now leaked, and its callback will be called repeatedly "forever". Javascript follows this important heuristic and so, although V8 allows us to intercept a Javascript value as it is garbage-collected via weak references, our code that deals with handles can also be made easier: *If the handle goes out of scope, the resource to which it refers is leaked.*

Armed with these heuristics, let's create our own handles with the help of NAN:
```C++
template <class T> class JSHandle {

    // This is the template from which all our instances will be created
    static Nan::Persistent<v8::FunctionTemplate> &theTemplate()
    {
        static Nan::Persistent<v8::FunctionTemplate> returnValue;

        // We only create the class the first time it's needed
        if (returnValue.IsEmpty()) {
            v8::Local<v8::FunctionTemplate> theTemplate =
                Nan::New<v8::FunctionTemplate>();

            // This is why we're using a C++ template class. The specific class
            // provides a static method named jsClassName() which we can use
            // to give our class a nice-looking name
            theTemplate
                ->SetClassName(Nan::New(T::jsClassName()).ToLocalChecked());

            // This is important. This is where we tell V8 to reserve one
            // slot for an arbitrary pointer.
            theTemplate->InstanceTemplate()->SetInternalFieldCount(1);

            // We also set the "displayName" property so our instances do not
            // simply appear as "Object" in the debugger
            Nan::Set(Nan::GetFunction(theTemplate).ToLocalChecked(),
                Nan::New("displayName").ToLocalChecked(),
                Nan::New(T::jsClassName()).ToLocalChecked());
            returnValue.Reset(theTemplate);
        }
        return returnValue;
    }
public:

    // This is the API we will use. In our bindings we will instantiate new
    // handles with HandleClassName::New(native_handle);
    static v8::Local<v8::Object> New(void *data)
    {
        v8::Local<v8::Object> returnValue =
            Nan::GetFunction(Nan::New(theTemplate())).ToLocalChecked()
            ->NewInstance();
        Nan::SetInternalFieldPointer(returnValue, 0, data);

        return returnValue;
    }

    // This is how we retrieve the pointer we've stored during instantiation.
    // If the object is not of the expected type, or if the pointer inside the
    // object has already been removed, then we must throw an error.
    static void *Resolve(v8::Local<v8::Object> jsObject)
    {
        void *returnValue = 0;

        if (Nan::New(theTemplate())->HasInstance(jsObject)) {
            returnValue = Nan::GetInternalFieldPointer(jsObject, 0);
        }
        if (!returnValue) {
            Nan::ThrowTypeError((std::string("Object is not of type ") +
                T::jsClassName()).c_str());
        }
        return returnValue;
    }
};
```

With this class template in a header file we can now create any number of Javascript classes that can be used to pass handles into Javascript land. For our example, we can declare the class
```C++
class JSRemoteResourceHandle : public JSHandle<RemoteResourceHandle> {
public:
	static const char *jsClassName() { return "RemoteResourceHandle"; }
};
```
and then we can use it to pass handles into Javascript:
```C++
RemoteResourceHandle *c_handle = remote_resource_retrieve(host, port);
Local<Object> jsRemoteResourceHandle = JSRemoteResourceHandle::New(c_handle);
```
We can now set ```jsRemoteResourceHandle``` as the return value for our binding. In Javascript, the handle will show up as an empty Javascript object. Of course we can, at our option, decorate the handle with properties as we would any Javascript object, but the important thing is that ```c_handle``` is now stored inside this Javascript object instance without it being accessible from Javascript.

Since "magic" pointers are important because they represent resources stored inside the native C API, and since it is impossible to create these special Javascript objects via Javascript, we can distinguish between a plain Javascript object and a special object which we have created previously. The ```::Resolve()``` method ensures that the object passed in is an instance of our special object class and throws a ```TypeError``` if it isn't. 

We can also render such a special Javascript object inert. For example, in our binding for ```remote_resource_release()``` we should have the following steps:
```C++
...
// We assert that jsHandle is a Javascript value of type Object which has been
// passed to our binding as its first parameter.
RemoteResourceHandle *c_handle = JSRemoteResourceHandle::Resolve(jsHandle);

// If the handle is NULL, that means the ::Resolve() method has already thrown
// an exception so we should return immediately.
if (!c_handle) {
	return;
}

remote_resource_release(c_handle);

// Render the special object inert. We set the internal pointer to NULL, thus
// causing any future calls to ::Resolve() to throw an exception. This is more
// aggressive than the way clearTimeout()/clearInterval() behave. Thus, you may
// wish to implement ::Resolve() such that it won't treat NULL as a cause for
// throwing an exception. Nevertheless, throwing an exception is not a bad way
// of handling the case of multiply releasing the same resource, because it
// informs developers of potential race conditions. Your call.
Nan::SetInternalFieldPointer(jsHandle, 0, 0);
...
```
Since these special objects can only be created using our bindings, we should avoid creating more than one Javascript object for a given native handle so as to avoid the case of stale pointers. The simple remote resource API shown above does not really give us the opportunity to make such a mistake. Let's consider the case where we have the opportunity to create multiple Javascript objects for the same handle, but where we must avoid doing so. Suppose we have an additional C API that has asynchronous completion:
```C
void remote_resource_set_int_async(RemoteResourceHandle resource, const char *key, int value,
	void (*completionCallback)(void *data, RemoteResourceHandle c_handle, bool wasSuccessful), void *data);
```
When the native API calls the callback we have provided it, it will, as a convenience, give us the ```c_handle``` as part of the context. Now, when the bindings, in turn, call the Javascript callback provided, if we are to maintain the one-to-one binding, we should pass a JSRemoteResourceHandle to the Javascript callback:
```C++
...
Local<Value> arguments[2] = {
	JSRemoteResourceHandle::New(c_handle),
	Nan::New(wasSuccessful)
};
jsCallback->Call(arguments, 2);
```
This is bad. We have just created a second Javascript handle containing the same "magic" pointer as the one we passed into Javascript land during our binding of ```remote_resource_retrieve()```. The way around this is to create a persistent reference to the Javascript handle in our binding for ```remote_resource_set_int_async()``` and pass it, along with the ```Nan::Callback```, which is a persistent reference to the Javascript function that we must call (```jsCallback``` in the example above), as the ```void *data``` parameter of the native callback. Read more about this in the <a href="#callbacks">callbacks</a> section.

## Callbacks
Any non-trivial C API will accept function pointers. The binding for such an API obviously accepts a Javascript function as one of its parameters. There is, at this point, a very important thing you have to keep in mind: C functions are "physical". That is, they are pieces of code which take up room on disk and in memory, and are stored in specially marked segments, marked as executable and protected from modification in all kinds of highly platform-dependent ways. In contrast, Javascript functions are merely pieces of data stored on the heap. Thus, Javascript functions can be created, copied, and destroyed at runtime whereas C functions can only be created at compile time. Thus, we cannot simply create a new C function at runtime to correspond to the Javascript function passed into our binding, to pass to the native API as a callback. Neither can we assume that exactly the same Javascript function will be passed to our binding every time it is called. We have no choice but to rely on a C programming artifact called a context or user data. In the above function ```remote_resource_set_int_async()``` it's the ```void *``` pointer which we pass to the API, and which we receive back from the API in the callback.

Most C APIs worth their salt will provide such a parameter. However, if you ever run into one that doesn't, all is not lost, although it has become far more complicated, going well beyond the scope of this reading. Suffice it to say that you can use [ffi][] or, more specifically, the C library it bundles under deps/libffi. Using this library, you can essentially create C functions at runtime such that a function you create
  0. has the signature required by the native API you are trying to bind,
  0. stores the context you need in an internal variable, and
  0. calls a function of your choosing with all the variables it receives from the native callback plus the context.

Another choice is to use trampoline from the GNU [libffcall][] library. Although trampoline looks much more elegant, it is not provided as an npm package, and its GPL v3 license may be difficult to reconcile with the more permissible licenses generally used with npm packages. The bottom line, however, is that if the native API does not provide room for a context, the only solutions left are fairly invasive and highly platfom-specific.

[ffi]: https://github.com/node-ffi/node-ffi
[libffcall]: https://www.gnu.org/software/libffcall/trampoline.html
