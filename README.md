# Coroutines
## What is?
A coroutine is a function that can suspend execution to be resumed later. 
Coroutines are stackless: they suspend execution by returning to the caller and the data that is required to resume execution is stored separately from the stack. This allows for sequential code that executes asynchronously, and also supports algorithms on lazy-computed infinite sequences and other uses.

A function is a coroutine if its definition does any of the following operators: co_return, co_yield amd co_await.

This is not a coroutine, is a normal function which prints "Hello world":

        void foo(){
            std::cout << "Hello world";
        }

        int main() {
            foo();    
        }

But this is not a coroutine either, it will cause a compile error.

        void foo(){
            co_return "Hello world"; //Neither with co_wait or co_yield
        }

        int main() {
            foo();    
        }

Every coroutine must have a return type that satisfies a number of requirements

## Elements of coroutines
Each coroutine is associated with the promise object, the coroutine handle and the coroutine state.
### Promise Object
This object is manipulated from inside the coroutine. The coroutine submits its result or exception through this object.
An example:

        struct return_object {
            struct promise_type {
                return_object get_return_object() { return {}; }
                std::suspend_never initial_suspend() { return {}; }
                std::suspend_never final_suspend() noexcept { return {}; }
                void unhandled_exception() {}               
                void return_void(){}
            };
        };

All the functions inside promise_type are mandatory, let´s explain each one.

- get_return_object() to obtain the object that is passed back to the caller.
- std::suspend_never initial_suspend() the coroutine keeps running until the first suspending co_await. This is the model for “hot-start” coroutines which execute synchronously during their construction and don’t return an object until the first suspension inside the function body.
- final_suspend after the coroutine function body has finished
When a coroutine reaches a suspension point the return object obtained earlier is returned to the caller/resumer, after implicit conversion to the return type of the coroutine, if necessary.
- return_value or return_void to define what the coroutine returns.
- if the coroutine ends with an uncaught exception, catches the exception and calls unhandled_exception() from within the catch-block
### Coroutine Handle
The coroutine handle, manipulated from outside the coroutine. This is used to resume execution of the coroutine or to destroy the coroutine frame.
An example:

        #include <concepts>
        #include <coroutine>
        #include <exception>
        #include <iostream>

        //definition of the return object and the promise type
        struct return_object {
            struct promise_type {
                return_object get_return_object() { return {}; }
                std::suspend_never initial_suspend() { return {}; }
                std::suspend_never final_suspend() noexcept { return {}; }
                void return_void() {}
                void unhandled_exception() {}
            };
        };

        //this is a coroutine, the return type satisfy the requirements and inside is co_return operator.
        return_object foo()
        {
            co_return; 
            /* destroys all variables with automatic storage duration in reverse order they were created.
            Calls promise_type.final_suspend() and co_awaits the result */           
        }

        int main()
        {
            //we create a handle of type coroutine_handle
            std::coroutine_handle<> handle;            
            //this coroutine do nothing
            foo();
            //we can manipulate the hanlde from outside the coroutine
            handle.resume();
            handle.destroy();
        }

The hanlde is like a pointer to the coroutine state, so we can change the value of any parameter in that state and the handle will remain the same. To do that we will use co_await but first we need to understand the coroutine states.
### Coroutine state
The coroutine state is an internal heap-allocated object that contains:
- the promise object 
- the parameters 
- the current suspension point
- local variables whose lifetime spans the current suspension point
## co_await
The unary operator co_await suspends a coroutine and returns control to the caller.
### Awaitable object
We must use the expression "co_await expr;" where "expr" is the awaitable object or awaiter. The awaiter has three methods:
- await_ready is an optimization, if it returns true, then co_await does not suspend the function. The <coroutine> header provides two pre-defined awaiters, std::suspend_always and std::suspend_never. As their names imply, suspend_always::await_ready always returns false, while suspend_never::await_ready always returns true. 
- await_suspend Store the coroutine handle every time await_suspend is called.
- await_resume returns the value of the co_await expression. 

An example:
    
        #include <concepts>
        #include <coroutine>
        #include <exception>
        #include <iostream>

        //definition of the return object and the promise type
        struct return_object {
            struct promise_type {
                return_object get_return_object() { return {}; }
                std::suspend_never initial_suspend() { return {}; }
                std::suspend_never final_suspend() noexcept { return {}; }
                void return_void() {}
                void unhandled_exception() {}
            };
        };

        struct awaiter {
            std::coroutine_handle<> *handle_;
            constexpr bool await_ready() const noexcept { return false; }
            void await_suspend(std::coroutine_handle<> handle) { *handle_ = handle; }
            constexpr void await_resume() const noexcept {}
        };

        //Coroutine using co_await
        return_object foo(std::coroutine_handle<> *handle)
        {
            //pass the handler to the await_suspend method
            awaiter wait{handle};
            for (int i = 1;; ++i) {
                co_await wait; //suspends the coroutine and returns control to the caller.
                std::cout << "It´s the " << i << "th time in coroutine" << std::endl;                
            }          
        }

        int main()
        {
            //we create a handle of type coroutine_handle
            std::coroutine_handle<> handle;            
            //pass the control of the handler to foo
            foo(&handle);
            for (int i = 1; i < 4; ++i) {
                std::cout << "It´s the " << i << "th time in main function" << std::endl;
                //handle() triggers one more iteration of the loop in counter
                handle();
            }
            //To avoid leaking memory, destroy coroutine state 
            handle.destroy();
        }

This is the output:
                                                                                        
        It´s the 1th time in main function
        It´s the 1th time in coroutine
        It´s the 2th time in main function
        It´s the 2th time in coroutine
        It´s the 3th time in main function
        It´s the 3th time in coroutine                                                                     
                                                                                        
You can avoid to use the awaitable by creating the handle in the return_object and returning it to the caller using the get_return_object method from promise_type.
An example:
                 
        #include <concepts>
        #include <coroutine>
        #include <exception>
        #include <iostream>

        struct return_object {
                struct promise_type {
                        return_object get_return_object() {
                                return {
                                        //return the handle
                                        .handle_ = std::coroutine_handle<promise_type>::from_promise(*this)
                                };
                        }
                        std::suspend_never initial_suspend() { return {}; }
                        std::suspend_never final_suspend() noexcept { return {}; }
                        void unhandled_exception() {}
                };
                //create the handle
                std::coroutine_handle<promise_type> handle_;
                operator std::coroutine_handle<promise_type>() const { return handle_; }
                operator std::coroutine_handle<>() const { return handle_; }
        };

        return_object foo()
        {
                for (int i = 1;; ++i) {
                        co_await std::suspend_always{}; 
                        std::cout << "It´s the " << i << "th time in coroutine" << std::endl;
                }          
        }

        int main()
        {
                //now we have to take the handler from foo
                std::coroutine_handle<> handle = foo();          
                for (int i = 1; i < 4; ++i) {
                        std::cout << "It´s the " << i << "th time in main function" << std::endl;
                        handle();
                }
                handle.destroy();
        }       
The output is the same.                                                                                
