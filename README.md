## Extending Python using C
Python is a high-level programming language that has many useful features. It is sought out by beginners in the coding industry thanks to its simple syntax, portability, and numerous external libraries. There are numerous other features that make the language prominent, one being extensibility. In computer science, extensibility is used in reference to being able to add more functionality to something, the Python language in this case. I'm going to focus specifically on the interaction between C and Python that can be used for extension. There are really two main reasons one might want to use C to extend Python:

 **1. Add functionality from C into Python:** 
 This typically consists of calling in C library functions or system calls that aren't available in Python. The `os` module is an example of this. 
 **2. Implement new built-in object types**
 C is significantly faster at runtime than Python, so it can be useful to write object classes in C, then instantiate and extend them within the original Python program. 

For this post, I'm going to focus on the former. We'll create a small module that wraps the C library function `strcmp()`, which is found in the `<string.h>` library. Here's a small example to demonstrate how `strcmp()` works:

    #include <stdio.h>
	#include <string.h>

	int main()
	{
	    char first[] = "hello world";
	    char second[] = "this is some text";
	    int result = strcmp(first, second);
	    
	    if (result == 0) {
	        printf("Strings are the same!\n");
	    }
	    else {
	        printf("Strings are not the same!\n");
	    }
	}
In essence, `strcmp()` compares two strings, and returns 0 if they are the same, and some non-zero integer if they are different. In our case here, when the program is run, the output is `Strings are not the same!`, which checks out. With our extension module, we want to be able to use this function in Python and retain the original functions functionality and speed. 

## Creating a Module
As mentioned before, extensions to Python are written in C programs called "modules". These modules use elements from the Python API to interact with the Python interpreter. We will want to name our module file something like `strmodule.c` or something memorable like that.
The first thing we want to do is connect the Python API to our module. You do this using the line:

    #include <Python.h>
    
This inclusion must be the first one you make in the module.
(note: On Macintosh computers, Python is a framework, so this line will be changed slightly to: `#include <Python/Python.h>` )
 
**"Wrapping" your Function**
Now you need to "wrap" the function you're adding to Python. Using some handy tools given to us in `Python.h`, this is fairly straightforward. The C version of your function will **always** come in one of the three following forms:

    // has some arguments
    static PyObject *function_name(PyObject *self, PyObject *args);
	
	// has arguments and keywords
    static PyObject *function_name(PyObject *self, PyObject *args, PyObject *kw); // with keywords
    
    // has no arguments
    static PyObject *function_name(PyObject *self);

The function name (denoted here as `*function_name`) is typically a combination of the module name and the function you're recreating. In our case, our function should be called: `*str_strcmp`. If you're familiar with C, you'll notice that the return type of these function is always `PyObject`, meaning it will return a Python object. Python doesn't have a `void` value like C does, so if our function doesn't return anything, we would need to include the line:

    Py_RETURN_NONE;

 which is a macro defined in the API. It serves as C's equivalent to Python's `None`. Back to arguments, `strcmp()` takes in two arguments: the two strings being compared. So we'll be following the format of the top-most function. 
 
With all this, we can start making a framework for our first method:
 

    static  PyObject *method_strcmp(PyObject *self, PyObject *args) {
		// function content here
		return; // we'll determine this
	};

Now we can start delving into what the makeup of the function will be.

To start, we need to initialize variables. For strcmp(), we have three variables:
`str1`, `str2` , and `result`. We initialize them in these lines:

    char *str1, *str2 = NULL;
    int result = -1; 
We don't want to start `result` at 0, because if something goes wrong, we don't want it returning 0 and thinking the function succeeded. Next we have to parse these arguments into local variables. We can do this using the function `PyArg_ParseTuple()`. In our method, we can use it like so:

    if (!PyArg_ParseTuple(args, "ss", &str1, &str2)) {
	    return NULL;
	}
This makes sure that the variables are correctly parsed, and otherwise returns null and the method won't continue running. The "ss" specifies that we want both str1 and str2 to be strings (or lists of characters technically). We don't need to parse `result` in this way because its our resulting value holder and aren't doing anything to it in the function.  
Now we can put in the main functionality of the method. The code is very simple, we can just run the function and give the return value to `result`:

    result = strcmp(str1, str2);

 And finally, we return the result back to the interpreter with: 

    return PyLong_FromLong(result);
    
specifying that we want to return a PyLongObject, which is equivalent to an integer object in Python. 
With all the functionality complete, here's how our primary method looks:

    static  PyObject *method_strcmp(PyObject *self, PyObject *args) {
		char *str1, *str2 = NULL;
	    int result = -1;

		if (!PyArg_ParseTuple(args, "ss", &str1, &str2)) {
		    return NULL;
		}
		result = strcmp(str1, str2);
		
		returnPyLong_FromLong(result);
	};

 ## Going Forward
 With your main functionality complete, you'd think you're done. Unfortunately, modules aren't that simple. You would need to write code for the definitions of your module, as well as the methods that it contains (I know, everything needs to be defined). Here are those two definitions:
 

    static  PyMethodDef  strcmpMethods[] = {
		{"strcmp", method_strcmp, METH_VARARGS, "Python interface for strcmp C library function"},
		{NULL, NULL, 0, NULL}
	};
	
    static  struct  PyModuleDef  strcmpModule = {
		PyModuleDef_HEAD_INIT,
		"strcmp",
		"Python interface for strcmp C library function",
		-1,
		strcmpMethods
	};
I'm not going to go in-depth and explain these as I did with the main method, however these are important to getting your module up and running.

What I do want to talk about, however, is why making these modules is important/useful. As I mentioned before, the main reasons a programmer would write a module to extend Python are to improve performance or to provide functionality that is unavailable in Python, but *is* available in C/C++. 

**Provide Functionality:**
In our example, we wanted to provide access to the `strcmp()` function. As I'm sure you know, you could just as easily code a simple version of this function in base Python:

    def strcmp(str1, str2):
	    if str1 == str2:
		    return 0;
		else:
			return 1;
This implementation is very simple and effective, so there's no practical need to code up an entire C module just for this. The aforementioned Python library `os` and another one, `ctypes` do this for C library functions. Libraries like these save programmers time in the long run, but sometimes if there aren't ones available for a given functionality, its easier to just hard-code it up yourself in C. 

**Performance Boost:**
In the working world, a programmer is bound to run into performance bottlenecks. Having the ability to jump into another language, like C, that is faster than the language you originally coded in is an essential tool to have in your arsenal. Python is an "interpreted" language, where as C is a "compiled" language. For those who don't know the difference, an interpreted language uses an interpreter to run programs. Interpreters are slow because they go line by line, executing each one and checking for errors. Compiled languages, on the other hand, use a compiler (as the name suggests). The compiler is significantly more effective at running through code, as it attempts to do it all at one, and spits out every error it found at the end. This is why typically in Python, if you get an error, you might only get one, like in the following error message:

    Traceback (most recent call last):
	  File "/Users/kfsullivan/Documents/example.py", line 6, in <module>
	    main()
	  File "/Users/kfsullivan/Documents/example.py", line 4, in main
	    print("%d\n", a / b)
	TypeError: unsupported operand type(s) for /: 'int' and 'str'
In a compiled language like C, you'll likely get multiple errors at once, which can sometimes be overwhelming:

    main.c: In function ‘main’:
	main.c:13:13: error: ‘b’ undeclared (first use in this function)
	     int a = b + c;
	             ^
	main.c:13:13: note: each undeclared identifier is reported only once for each function it appears in
	main.c:13:17: error: ‘c’ undeclared (first use in this function)
	     int a = b + c;
	                 ^
Anyways, the point I'm trying to get across is: the fact that compiled languages like C use a compiler and go over the entire file at once are much faster than interpreted ones like Python, and thus if a programmer happens to be familiar with both, can take advantage of this performance difference.

In the end, programming is all about solving the given task in the most efficient way possible. That's why it's amazing that some languages like Python allow for extension by other language. With tools like these available at a capable programmer's fingertips, they can in theory solve any problem thrown at them. Efficiency is usually priority number one, and Python's extensibility is one reason that it remains my favorite programming language. 

## Further Readings:

 - [Medium - Execute Python at the speed of C - Extending Python](https://medium.com/practo-engineering/execute-python-code-at-the-speed-of-c-extending-python-93e081b53f04#:~:text=Python%20enables%20faster%20development%20with,operations%20into%20an%20extension%20module.)
 - [Python Docs - Extending Python with C or C++](https://docs.python.org/3/extending/extending.html#writing-extensions-in-c)
