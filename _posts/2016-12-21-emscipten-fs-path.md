---
title:  "Emscriptened!"
categories: [C++, C++17, filesystem]
tags: [C++, C++17, filesystem]
---
The `path` to the web. 

![img](../../assets/path2.jpg)
In my [last post](https://adishavit.github.io/2016/fs-path/) I showed the effects of various methods of `filesystem::path`. It occurred to me that it might be a fun little project to generate a web-based interactive version of the post.  

Since I didn't want to reimplement `filesystem::path` in Javascript, this would be a good chance to try [emscripten](https://kripken.github.io/emscripten-site/index.html), something that I've been meaning to do for some time:  

> *Emscripten is an LLVM-based project that compiles C and C++ into highly-optimizable JavaScript in asm.js format. This lets you run C and C++ on the web at near-native speed, without plugins.*
 
I needed emscripten to use a compiler and standard library that supports `std::filesystem::path`. Unfortunately, this is not the case and the current version of the emscripten compiler and library do not support `std::filesystem::path` nor `std::experimental::filesystem::path`. 

Since `std::filesystem` is based on [The Boost Filesystem Library](http://www.boost.org/doc/libs/1_62_0/libs/filesystem/doc/reference.html), perhaps I could emulate  `std::filesystem` using an emscriptened version of Boost.

Fortunately, the [chronotext-boost](https://github.com/arielm/chronotext-boost) project provides wide cross-platform support for Boost (1.58) on iOS, Android and Emscripten (to name a few). The [Wiki](https://github.com/arielm/chronotext-boost/wiki) is relatively straight forward, and using [emscripten portable](https://kripken.github.io/emscripten-site/docs/getting_started/downloads.html) I had the emscriptened Boost samples running in the browser pretty quickly.

### Embinding
Since I only wanted to expose a few methods of `std::filesystem::path`, I chose the [embind](https://kripken.github.io/emscripten-site/docs/porting/connecting_cpp_and_javascript/embind.html) approach.  
Here is the full C++ code:  

```cpp
#include <string>
#include <emscripten/bind.h>
#include <boost/filesystem.hpp>

using namespace emscripten;
namespace fs = boost::filesystem;

#define PATH_FUNC(func_name)                  \
std::wstring func_name(std::wstring const& p) \ 
{ return fs::path(p).func_name().wstring(); }   

PATH_FUNC(root_name);
PATH_FUNC(root_directory);
PATH_FUNC(root_path);
PATH_FUNC(relative_path);
PATH_FUNC(parent_path);
PATH_FUNC(filename);
PATH_FUNC(stem);
PATH_FUNC(extension);
PATH_FUNC(remove_filename); // in this context, remove_filename is equivalent to parent_path()

// register exported functions
EMSCRIPTEN_BINDINGS(fs_module)
{
  function("root_name", &root_name);
  function("root_directory", &root_directory);
  function("root_path", &root_path);
  function("relative_path", &relative_path);
  function("parent_path", &parent_path);
  function("filename", &filename);
  function("stem", &stem);
  function("extension", &extension);
  function("remove_filename", &remove_filename);
}
```

#### Code Considerations
As seen in the code, the functions are all stateless (and macro-generated identically), taking in a string and returning a string. Conversions to and from `std::filesystem::path` are done inside the functions so that `std::filesystem::path` instances are not exported or exposed.

I decided to go with stand-alone functions since wrapping `std::filesystem::path` with another class, or even exporting it itself, would require the user on the JS side to `new` the class and then remember to `.delete()` it. Access to the functions on the Javascript side is done via e.g. `Module.filename()` so it still looks and feels like method access *sans* the memory management.  

I used `std::wstring` since this allows me to seamlessly and natively support Javascript's Unicode strings.

Note also that the Boost version of `boost::filesystem::path` built with emscripten is a Linux based one and ths gives slightly different results with OS-specific things like root names. So although the Windows version will identify `C:\` as the root path, this Linuxy-build version will not. 

Throw in a little JavaScript and I give you **Empath** the emscriptened `path`:

### Empath
Enter your path here and behold the magic of C++ and Javascript!   
Path: <input id="path" type="text" name="firstname"> 
<p id="output"></p>

<script src="../../assets/scripts/fs_path.js">
</script>
<script>
	var display_path_info = function(path) {
	  console.log('Path: ' + path);
	
	  var methods = [ 
	                  'filename', 'stem', 'extension',
	                  'root_name', 'root_directory', 'root_path',
	                  'relative_path', 'parent_path',
	                  'remove_filename'
	                ];
	
	  var res = '<pre><table style="width:100%">';
	  methods.forEach(function(m){
	    res = res + '<tr><td>'+ m + '()</td><td>' + Module[m](path) + '</td></tr>'; 
	  });
	  res = res + '</table></pre>';
	
	  return res; 
	}
	var path_element = document.getElementById("path");
	
	path_element.defaultValue = "//home/folder/foo.bar";
	var get_path_info = function() {
	    document.getElementById("output").innerHTML = display_path_info(path_element.value);
	};
	get_path_info(); // call it for default value
	
	// set event listeners
	path_element.addEventListener("keydown", get_path_info);
	path_element.addEventListener("keyup", get_path_info);
	path_element.addEventListener("keypress", get_path_info);
	path_element.addEventListener("change", get_path_info);
	path_element.addEventListener("drop", get_path_info);
	path_element.addEventListener("paste", get_path_info);
</script>

Newer versions of emscripten will also generate WebAssembly, which is gaining support with newer versions of most browsers. This makes the web another target platform for C++ that should not be overlooked when performance is at a premium.

I ❤ your feedback so do use the comments below, Twitter or Reddit.

*Acknowledgments:
[banner](https://www.flickr.com/photos/gold41/6045175765/) :: 
[chronotext-boost](https://github.com/arielm/chronotext-boost)*
