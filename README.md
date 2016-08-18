# Writing Node.js Bindings - General Principles

This material is about creating Node.js interfaces for native libraries written in C. It is not so much about the mechanics of writing the code, or about the structure of a npm package containing a native addon, but about the various situations you are likely to face when writing addons. It starts with the assumption that it is best to create an addon that provides as little abstraction as possible so as to allow you to provide a Javascript API to the consumers of your project that is itself written in Javascript. The portion of the Node.js documentation that describes native addons, the V8 reference, and the reference provided by the Native Abstractions for Node ([NAN][]) project give you an ample toolset for creating native addons. Once you've managed to create your first native addon, and are ready to bring a complete native library into the Node.js world, you will be faced with having to translate the artifacts of the C language into Javascript concepts.  At that point, it pays to be systematic. It is beneficial to break the library's API into constants, enums, structures, and functions, and bind each onto the Node.js module you are creating in as automated a fashion as possible. In this material, I will describe some patterns for dealing with data structures, pointers, and callbacks one needs to pass to the native library.

# Assumption

It is best to write as little logic as possible into the bindings themselves. Ideally the addon is a one-to-one mapping of the underlying native API itself. Convenient abstractions can then be implemented in Javascript.

Nevertheless, the underlying language has artifacts that need not be mapped.

  0. For example, C has ```void *user_data``` with its function pointers, but that's only to provide room for context. Javascript is the king of context, so you don't need to map this mechanism into Javascript. OTOH, you need this mechanism for the <a href="#callbacks">internals of the bindings</a>.
  0. C also doesn't have a concept of variable-length arrays. Thus, many APIs that accept variable-length arrays do so by accepting two parameters: a pointer to the data, and an integer indicating how many items are stored at the pointer. For example:

    ```C
    void draw_multi_line(point *points, size_t point_count);
    ```

    or

    ```C
    struct point_array {
      point *points;
      size_t point_count;
    };
    ```

    Since Javascript has a native array type, it is unnecessary to write the bindings in such a way that you have to pass the lengths from Javascript to the binding, otherwise you invariably end up with

    ```JS
    myAddon.draw_multi_line( points, points.length );
    ```

    or

    ```JS
    {
      points: points,
      point_count: points.length
    } 
    ```

    which is superfluous, since the array already conveys its length to the C++ bindings.
  0. You might also consider pointers that are filled out by the native side. In that case you can have the binding accept an empty object the properties of which it fills out (a receptacle) or you can return a new object you create inside the binding. But what if the native function additionally returns a result code (like ```errno``` for example)? Do you retain one-to-one-ness and use a receptacle object or do you break one-to-one-ness and return an object upon success and an error code otherwise? Do you throw the error code as an exception (again, breaking one-to-one-ness)? Your call.

# General principles

  0. Trust data from the native side, but validate data from the JS side. Do not directly cast a JS value to the type needed by the native API. Instead check that it contains the type needed by the native API first. If it does, cast it and proceed. If it doesn't, throw a TypeError with text naming the parameter that failed validation and return immediately.
  0. Avoid argument juggling. IOW, enforce that there be a certain number of arguments and that each argument have a certain type. At most, allow an argument to optionally be ```null``` (especially for strings).
  0. Make the portion of the stack that starts with the binding mimic the JS behaviour of throwing an exception when needed and tearing down afterwards. This means that the binding only sets the return value if no exceptions were thrown, otherwise it returns early. This in turn means that functions called by the binding (except for the underlying native API, the signature of which we cannot influence) have to return a boolean indicating whether the binding is to immediately return. Internally, these helper functions perform the setup required for later calling the native API and, if validation or other errors occur in the process, they throw an exception and ```return false;```. The fact that the binding checks the return value only for the purposes of deciding whether to return immediately rather than using the return value to decide whether to throw an exception and then return is a design choice. The idea is to be similar to JS in that we throw the exception nearest to the point of failure, rather than returning ```false``` and expecting the caller to throw the exception.

    ```C++
    // TODO: Add examples for each case, highlighting the tradeoff between
	// error message specificity (what failed) and error location specificity
	// (what were you trying to do when it failed)
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
 
Exceptions are only thrown if it turns out that the length of the argument list is not equal to two, or if the arguments do not contain values that can be cast as doubles. The return value is constructed from the return value of the native API.

## Data
This is any kind of data that does not fit into a primitive value that the language can handle directly. The distinction from primitive types is that now we're dealing with pointers. There are many ways in which data can be dealt with, so let's break up the use cases and look at each in turn.
  0. The structure. This is a pointer to a well-known structure into which the bindings can decend and from which they can construct a Javascript object. The structure is either passed to the native library or the native library may return it.
  0. The handle. This is a "magic" pointer that is usually bookended by two API calls - one to create it, and one to destroy it.

### The simple case - a transparent structure
This is an API of the form:
```C
struct coordinates {
  double latitude;
  double longitude;
  double altitude;
};
/* Both APIs return true on success */
bool find_restaurant_by_name(const char *name, struct coordinates *location);
bool restaurant_at_coordinates(const struct coordinates *location);
```

The structure of type ```coordinates``` can be converted to/from Javascript without allocating any memory and worrying about who owns the pointers. In the binding for ```find_restaurant_by_name()``` we accept an empty JS object and fill it in from the pointer returned by the API if the API returns ```true```. In either case we forward the return value we receive from the API.

In the case of the second API, the binding accepts a Javascript object which has properties named after the members of the ```cordinates``` structure. We declare a local C structure, retrieve each property of the object passed in, assert that it is of the correct type, throwing an exception and returning immediately if it isn't, and casting the value as a double, while assigning it to the corresponding C structure member.

In general, it's a good idea to create a pair of helper functions that will convert the C structure to a JS structure and back. In [NAN][] parlance, this is what that looks like:
```C++
bool c_coordinates(v8::Local<v8::Object> jsCoordinates,
		struct coordinates *location) {

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

Local<Object> js_coordinates(const struct coordinates *location,
		Local<Object> destination = Nan::New<Object>()) {
  destination->Set(Nan::New("latitude").ToLocalChecked(),
    Nan::New(location->latitude));
  destination->Set(Nan::New("longitude").ToLocalChecked(),
    Nan::New(location->longitude));
  destination->Set(Nan::New("altitude").ToLocalChecked(),
    Nan::New(location->altitude));
  return destination;
}
```
The function that converts to Javascript works with an existing object if the second parameter is specified, but will create a new object if it is not. This comes in handy when you later use this function with <a href="#callbacks">callbacks</a>.

You can maximize code reuse and increase the readability of the ```c_<structurename>()``` function if you create a macro for performing the validation. For example,
```C++
#define VALIDATE_VALUE_TYPE(value, typecheck, message, failReturn) \
    if (!(value)->typecheck()) { \
        Nan::ThrowTypeError(message " must satisfy " #typecheck "()"); \
        return failReturn; \
    }
```

### The complex case - handles

In this case the underlying API gives you a "magic" pointer that identifies a resource the API maintains internally. The actual structure associated with the pointer type may or may not be opaque, but the value of the pointer is important. Note that by its very nature, such an API has to provide at least two functions: one for obtaining the "magic" pointer, and one for informing the API that it is no longer needed and may be disposed of. Here's an example API:

```C
typedef void *RemoteResourceHandle;

RemoteResourceHandle remote_resource_retrieve(const char *host, uint16_t port);
bool remote_resource_set_int(RemoteResourceHandle resource, const char *key, int value);
bool remote_resource_release(RemoteResourceHandle c_handle);
```

How do we bind this API? We could do something simple and treat the handle as an array of bytes. We don't even need to know whether pointers are 32 bit values or 64 bit values. We simply create an array of length ```sizeof(RemoteResourceHandle)``` and store the pointer as an array of bytes. We can even create functions

```C++
Local<Array> js_RemoteResourceHandle(RemoteResourceHandle handle, Local<Array> destination = Nan::New<Array>(sizeof(RemoteResourceHandle)));
bool c_RemoteResourceHandle(Local<Array> jsHandle, RemoteResourceHandle *c_handle);
```

and basically pass the pointer blindly into Javascript land. This is not a very good solution since we have chosen not to trust anything coming from the Javascript side, and we have no way of verifying that the integrity of the pointer value is preserved when we get it back in the binding for ```remote_resource_set_int()``` and ```remote_resource_release()```. It would be really nice if we could create a Javascript object and assign a property to it which is not visible in Javascript land. Fortunately, Javascript allows us to create new types of objects, and V8 exposes this functionality to the native side as well, with the important native-only capability of assigning a number of arbitrary pointers to an object instance.

Before diving into the process of passing a handle into Javascript we should consider an important aspect of handles. Whenever we store the value of the handle, we risk creating a copy that will become stale. That is, for a given handle, we risk passing it to ```remote_resource_set_int()``` after having already called ```remote_resource_release()```. This is especially true in Javascript code, which can be highly asynchronous, in turn making the tracking of the sequence in which the above APIs get called a fairly difficult task. Fortunately, Javascript itself provides us with an excellent template for dealing with handles. Consider ```setTimeout()```:

  0. 
  
  ```JS
  var x = setTimeout( function() {}, 1000 );
  
  // What exactly does x now hold?
  ```
  The function returns a "magic" value, which is completely useless for anything other than passing it to ```clearTimeout()```. Not even the type of the value is certain. In browsers, it's usually an integer, however, in Node.js it's an object. All we know is that it can be stored in a variable, and can be copied from variable to variable.
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
  If we overwrite all references to the "magic" value, we have no way of removing the timeout. In terms of memory management this is not a big poblem in the case of a timeout, because its semantics are such that it will simply run once and then be removed implicitly. However, in the case of ```setInterval()```, overwriting the return value of the function means that the interval has now leaked, and its callback will be called repeatedly "forever", keeping the memory allocated for it in place. Javascript follows this important heuristic and so, although V8 allows us to intercept a Javascript value as it is garbage-collected via weak references, our code that deals with handles can also be made easier: *If the handle goes out of scope before it is released, the resource to which it refers is leaked.*

Armed with these heuristics, let's create our own handles with the help of NAN:<a name="handle-class-template"></a>
```C++
template <class T> class JSHandle {

    // This is the V8 template from which all our instances will be created
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
    // object has already been removed, then we must throw an error. Note that
	// we assume that a Javascript handle containing a null pointer is not a
	// valid Javascript handle at all, so this setup is not adequate for
	// wrapping NULL handles.
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
class JSRemoteResourceHandle : public JSHandle<JSRemoteResourceHandle> {
public:
	static const char *jsClassName() { return "RemoteResourceHandle"; }
};
```
and then we can use it to pass handles into Javascript:
```C++
RemoteResourceHandle *c_handle = remote_resource_retrieve(host, port);
...
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

bool returnValue = remote_resource_release(c_handle);

if (returnValue) {

	// Render the special object inert. We set the internal pointer to NULL,
	// thus causing any future calls to ::Resolve() to throw an exception. This
	// is more aggressive than the way clearTimeout()/clearInterval() behave.
	// Thus, you may wish to implement ::Resolve() such that it won't treat
	// NULL as a cause for throwing an exception. Nevertheless, throwing an
	// exception is not a bad way of handling the case of multiply releasing
	// the same resource, because it informs developers of potential race
	// conditions. Your call.
	Nan::SetInternalFieldPointer(jsHandle, 0, 0);
}

info.GetReturnValue().Set(Nan::New(returnValue));
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
The first argument to the Javascript callback is a handle we have created for this one function call. However, conceptually, the c_handle is the same as the one for which we have previously created a Javascript wrapper. This is bad. We have just created a second Javascript handle containing the same "magic" pointer as the one we passed into Javascript land during our binding of ```remote_resource_retrieve()```. The reason this is bad is that when the Javascript code invalidates one of the handles, the underlying C API in turn invalidates its own handle. However, we still have a copy of the now stale handle in a second Javascript handle which remains perfectly valid. This introduces a discrepancy between the state of the bindings and the state of the C library.

The way around this is to create a persistent reference to the Javascript handle in our binding for ```remote_resource_set_int_async()``` and pass it, along with the ```Nan::Callback```, which is a persistent reference to the Javascript function that we must call (```jsCallback``` in the example above), as the ```void *data``` parameter of the native callback. Read more about this in the <a href="#callbacks">callbacks</a> section.

## Callbacks
Any non-trivial C API will offer functions which accept function pointers. The binding for such an API obviously accepts a Javascript function as one of its parameters. There is, at this point, a very important thing you have to keep in mind: C functions are "physical". That is, they are pieces of code which take up room on disk and in memory, and are stored in specially marked segments, marked as executable and protected from modification in all kinds of highly platform-dependent ways. In contrast, Javascript functions are merely pieces of data stored on the heap. Thus, Javascript functions can be created, copied, and destroyed at runtime whereas C functions can only be created at compile time. Thus, we cannot simply create a new C function at runtime to correspond to the Javascript function passed into our binding, to pass to the native API as a callback. Neither can we assume that exactly the same Javascript function will be passed to our binding every time it is called. We have no choice but to rely on a C programming artifact called a context or user data. In the above function ```remote_resource_set_int_async()``` it's the ```void *``` pointer which we pass to the API, and which we receive back from the API in the callback.

Most C APIs worth their salt will provide such a parameter. However, if you ever run into one that doesn't, all is not lost, although it will have become far more complicated, going well beyond the scope of this reading. Suffice it to say that you can use [ffi][] or, more specifically, the C library it bundles under ```deps/libffi```. Using this library, you can essentially create C functions at runtime such that a function you create
  0. has the signature required by the native API you are trying to bind,
  0. stores the context you need in an internal variable, and
  0. calls a function of your choosing with all the variables it receives from the native callback plus the context.

Another choice is to use trampoline from the GNU [libffcall][] library. Although trampoline looks much more elegant, it is not provided as an npm package, and its GPL v3 license may be difficult to reconcile with the more permissive licenses generally used with npm packages. The bottom line, however, is that if the native API does not provide room for a context, the only solutions left are fairly invasive and highly platfom-specific. Thus, in the remainder of this material we will only deal with examples of native APIs that provide room for a context parameter.

Whenever a native API accepts a callback function pointer, it usually uses it in one of two ways:
  0. The callback has a finite use and is removed implicitly as part of the last one of its executions.
  0. The callback is finite but open-ended - that is, it remains in place until it is removed by a call to another API or by a second call to the same API.

The difference between the two is that in the latter case we must pass all the information needed to remove the callback to Javascript so that we may receive it back in the binding to the teardown API, whereas in the implicit case we need only pass that information to the callback which we give to the native API via its ```void *``` context parameter. Let's have a look at each lifecycle in turn.

### The implicitly removed callback
We've already seen an example of an implicitly removed callback: ```remote_resource_set_int_async()```. We attach the callback in the binding and the underlying API removes the callback after it calls it once. This is the general modus operandi of completion notification callbacks. Let's look at some of the essential parts of the binding:
```C++
...
// At the top of the file we define a structure for tying together the data
// for the completion callback
struct CallbackParams {
	Nan::Callback *jsCallback;
	Persistent<Object> *jsHandle;
};
...
// This is the callback function we will pass to the underlying native API. The
// context parameter identifies the JS callback we need to call, and we have
// all the information we need for constructing the JS callback's arguments.
void setIntCallback(void *data, RemoteResourceHandle c_handle, bool wasSuccessful) {

	// Since the stack containing this function call starts with the native
	// library we must make sure that we have a V8 scope in which to create new
	// Javascript variables
	Nan::HandleScope scope;

	CallbackParams *params = (CallbackParams *)data;
	Local<Value> arguments[2] = {

		// We do not use JSRemoteResourceHandle::New(c_handle) here because we
		// want to avoid creating two different JS objects referring to the
		// same native handle - see the section on handles.
		Nan::New<Object>(*(params->jsHandle)),
		Nan::New(wasSuccessful)
	};

	// Call the callback
	params->jsCallback->Call(arguments, 2);

	// Since we know that this callback will never be called again, we free its
	// context
	delete params->jsCallback;
	params->jsHandle->Reset();
	delete params->jsHandle;
	delete params;
}
...
// In the binding we retrieve the native handle from the JS handle - as
// described in the section on handles
RemoteResourceHandle c_handle = JSRemoteResourceHandle::Resolve(jsHandle);
if (!c_handle) {
	return;
}
...
struct CallbackParams *params = new struct CallbackParams;
if (!params) {
	return;
}
...
// Create a pointer to a Nan::Callback - a persistent reference to the JS
// function. This is a pointer we can pass to the native callback.
params->jsCallback = new Nan::Callback(jsCallback);
if (!params->jsCallback) {
	delete params;
	return;
}

// Create a pointer to a persistent reference to the JS handle so we can pass
// it back to JS from the callback
params->jsHandle = new Nan::Persistent<Object>(jsHandle);
if (!params->jsHandle) {
	delete params->jsCallback;
	delete params;
	return;
}

// Now we have everything for the call to the native API
remote_resource_set_int_async(c_handle, key, value, setIntCallback, params);
...
```

### The explicitly removed callback
This setup most resembles Javascript's own ```setInterval()```/```clearInterval()``` pair. The semantics are that the first API receives a function that will be called periodically whenever a relevant event occurs, and returns a handle that can be passed to a second API which will cause the library to release the resources associated with periodically calling the function.

Below is an example of such an API pair.

```C
FileSystemWatch watch_file_name(const char *file_name,
	void (*file_changed)(void *context, const char *file_name, const enum
		FileEvent event),
	void *context);

bool unwatch_file_name(FileSystemWatch watch);
```

A variation of this API is one where the callback returns a boolean value to indicate to the native library whether it should keep calling it or whether it should release the resources associated with calling it.

```C
FileSystemWatch watch_file_name(const char *file_name,
	bool (*file_changed)(void *context, const char *file_name,
		const enum FileEvent event),
	void *context);

bool unwatch_file_name(FileSystemWatch watch);
```

In both cases we must construct the Javascript handle in such a way that it contains all the information necessary for later removing it. To implement our binding, we must first create a structure for the context and a C function we will use as the callback we pass to the C API.

```C++
struct FileSystemWatchCallbackData {

	// The native handle
	FileSystemWatch watch;

	// A persistent reference to the Javascript callback associated with the
	// native handle
	Nan::Callback *callback;
};

void file_changed_callback(void *context, const char *file_name,
		const enum FileEvent event) {

	// Since the stack containing this function call starts with the native
	// library we must make sure that we have a V8 scope in which to create new
	// Javascript variables
	Nan::HandleScope scope;

	// Cast the context back to a structure pointer
	FileSystemWatchCallbackData *data =
		(FileSystemWatchCallbackData *)context;

	// Construct the arguments for the call to Javascript
	Local<Value> arguments[2] = {
		Nan::New(file_name).ToLocalChecked(),
		Nan::New((int)event);
	};

	// Call the Javascript callback
	data->callback->Call(2, arguments);
}
```
We then define a type of handle which will correspond to the native FileSystemWatch and which will bear its name in Javascript. Our handle will store a pointer to a ```FileSystemWatchCallbackData``` structure. We will use the handle definition from <a href="#handles">handles</a>.
```C++
class JSFileSystemWatch: public JSHandle<JSFileSystemWatch> {
public:
	const char *JSClassName() { return "FileSystemWatch"; }
};
```
The binding for our native API, including the validation, then looks like this:
```C++
NAN_METHOD(bind_watch_file_name) {

	// Make sure the arguments we receive are as expected
	VALIDATE_ARGUMENT_COUNT(info, 2);
	VALIDATE_ARGUMENT_TYPE(info, 0, IsString);
	VALIDATE_ARGUMENT_TYPE(info, 1, IsFunction);

	// Allocate the structure which will hold everything we need
	FileSystemWatchCallbackData *callbackData =
		new FileSystemWatchCallbackData;

	// Create a persistent reference to the Javascript function object using
	// NAN's handy Nan::Callback API
	callbackData->callback = new Nan::Callback(Local<Function>::Cast(info[1]));

	// Call the native API
	callbackData->watch = watch_file_name(
		(const char *)*String::Utf8Value(info[0]),
		file_changed_callback,
		callbackData);

	// The native C API failed. Clean up and return.
	if (!callbackData->watch) {
		delete callbackData->callback;
		delete callbackData;
		info.GetReturnValue().Set(Nan::Null());
		return;
	}

	// Wrap the structure in a handle and return the handle to Javascript
	info.GetReturnValue().Set(JSFileSystemWatch::New(callbackData));
}
```
The binding for the API to release the ```FileSystemWatch``` handle then looks like this:
```C++
NAN_METHOD(bind_unwatch_file_name) {

	// Make sure the arguments we receive are as expected
	VALIDATE_ARGUMENT_COUNT(info, 1);
	VALIDATE_ARGUMENT_TYPE(info, 0, IsObject);

	// Cast the first argument as a Javascript object
	Local<Object> jsWatch = Nan::To<Object>(info[0]).ToLocalChecked();

	// Retrieve the native structure we created in the other binding
	FileSystemWatchCallbackData *callbackData =
		(FileSystemWatchCallbackData *)JSFileSystemWatch::Resolve(jsWatch);

	// JSFileSystemWatch::Resolve() has failed and it has thrown an appropriate
	// exception. Thus, we can return immediately.
	if (!callbackData) {
		return;
	}

	// Call the native API and store the return value
	bool returnValue = unwatch_file_name(callbackData->watch);

	// Removing the native FileSystemWatch has failed, so we reflect that in
	// our return value, and we return without freeing our structure or the
	// Javascript handle.
	if (!returnValue) {
		info.GetReturnValue().Set(Nan::New(returnValue));
		return;
	}

	// Free the reference to the Javascript function object because it will not
	// be called anymore
	delete callbackData->callback;

	// Delete the structure itself
	delete callbackData;

	// Invalidate the Javascript handle
	Nan::SetInternalFieldPointer(jsWatch, 0, 0);
}
```
### Reentrancy problems
In the case of a hybrid API, where the callback can be removed both implicitly by returning ```false``` from it, and explicitly, by calling the API to remove it (```unwatch_file_name()``` in the above example), we run into a complication where the Javascript implementation of the callback can call the binding for ```unwatch_file_name()```, but we must also free the same handle and structure from the native callback in case the Javascript callback returns indicating that it is to be implicitly removed. This raises the problem that we may accidentally end up freeing the same structure twice. To illustrate, consider the hybrid version of the above API:

```C
FileSystemWatch watch_file_name(const char *file_name,
	bool (*file_changed)(void *context, const char *file_name, const enum
		FileEvent event),
	void *context);

bool unwatch_file_name(FileSystemWatch watch);
```
The only difference is that the return value of the callback is boolean rather than void. The semantics are that, if the callback returns ```false```, the callback is implicitly removed and the structure and the handle are to be freed. However, we must keep in mind that, as we call the Javascript callback from the native callback, the body of the Javascript callback might include a call to the API to explicitly release the file system watch. For example, we might have
```JS
var watchResolvConf = myLibrary.watch_file_name("/etc/resolv.conf", function( fileName, fileSystemEvent ) {

	// Do something ...

	// Explicitly free the handle
	myLibrary.unwatch_file_name( watchResolvConf );

	// Do some more stuff ...

	// Implicitly free the handle
	return false;
} );
```
This is a contrived example, however, as the Javascript code becomes more complex, it is possible that it accidentally attempts to free the handle twice - once via the explicit API call made from within the callback, and once implicitly via the ```false``` return value. This is a mistake, of course, but we must handle it to the extent that we should avoid a double free and a resulting segmentation fault. To do so, we must modify the structure ```FileSystemWatchCallbackData``` to also contain a persistent reference to the Javascript handle. This results in a somewhat self-referential structure, in that the structure contains a persistent reference to the Javascript handle which, in turn, contains a pointer to the structure. The structure and the callback will then have the following code:
```C++
struct FileSystemWatchCallbackData {

	// The native handle
	FileSystemWatch watch;

	// A persistent reference to the Javascript callback associated with the
	// native handle
	Nan::Callback *callback;

	// A persistent reference to the Javascript handle
	Nan::Persistent<Object> *jsHandle;
};

bool file_changed_callback(void *context, const char *file_name,
		const enum FileEvent event) {

	// Since the stack containing this function call starts with the native
	// library we must make sure that we have a V8 scope in which to create new
	// Javascript variables
	Nan::HandleScope scope;

	// Cast the context back to a structure pointer
	FileSystemWatchCallbackData *data =
		(FileSystemWatchCallbackData *)context;

	// Construct the arguments for the call to Javascript
	Local<Value> arguments[2] = {
		Nan::New(file_name).ToLocalChecked(),
		Nan::New((int)event);
	};

	// Before calling the Javascript callback, we retain a local reference to
	// the Javascript handle, because the callback may contain code which
	// invalidates the handle, and, if it returns false, we must also
	// invalidate the handle, but only if it is still valid. Thus, we need to
	// be able to check if the handle is still valid after the callback.
	Local<Object> jsHandle = Nan::New<Object>(*(context->jsHandle));

	// Call the Javascript callback
	Local<Value> jsReturnValue = data->callback->Call(2, arguments);

	// If the return value is not a boolean, throw an exception and retain
	// this callback
	VALIDATE_CALLBACK_RETURN_VALUE_TYPE(jsReturnValue, IsBoolean,
		"file_changed_callback", true);

	// The return value is a boolean. Now, let's convert it to a native bool.
	bool returnValue = Nan::To<bool>(jsReturnValue).FromJust();

	// A false return value indicates that this callback context is to be freed
	// implicitly. However, it may have already been freed explicitly in the
	// course of the above Javascript callback. So, if we are asked to free
	// implicitly here, we must first ascertain that the context is still valid.
	if (!returnValue) {

		// Ascertain that the callback context remains valid. This will throw
		// an exception if the handle has been invalidated as part of an
		// explicit release via a call to the unwatch_file_name() binding.
		data = (FileSystemWatchCallbackData *)
			JSFileSystemWatch::Resolve(jsHandle);

		// If the data is still around then we free it here, implicitly.
		if (data) {
			delete data->callback;
			data->jsHandle->Reset();
			delete data->jsHandle;
			delete data;
		}
	}

	return returnValue;
}
```

The above code will throw an exception if the Javascript code attempts to doubly release the handle. If you want to be less drastic, and more like ```clearTimeout()```, you can add a method to ```JSHandle``` that will not throw an exception if the pointer stored within the Javascript object is ```NULL``` and then use that method instead of ```::Resolve()``` in the above code.

[ffi]: https://github.com/node-ffi/node-ffi
[libffcall]: https://www.gnu.org/software/libffcall/trampoline.html
[NAN]: https://github.com/nodejs/nan/
