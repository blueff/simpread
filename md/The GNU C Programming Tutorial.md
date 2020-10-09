\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[crasseux.com\](http://crasseux.com/books/ctutorial/argc-and-argv.html)

Node:argc and argv, Next:[Processing command-line options](http://crasseux.com/books/ctutorial/Processing-command-line-options.html#Processing%20command-line%20options), Previous:[Putting a program together](http://crasseux.com/books/ctutorial/Putting-a-program-together.html#Putting%20a%20program%20together), Up:[Putting a program together](http://crasseux.com/books/ctutorial/Putting-a-program-together.html#Putting%20a%20program%20together)

### `argc` and `argv`

So far, all the programs we have written can be run with a single command. For example, if we compile an executable called `myprog`, we can run it from within the same directory with the following command at the GNU/Linux command line:  

```
./myprog
```

However, what if you want to pass information from the command line to the program you are running? Consider a more complex program like GCC. To compile the hypothetical `myprog` executable, we type something like the following at the command line:  

```
gcc -o myprog myprog.c
```

The character strings `-o`, `myprog`, and `myprog.c` are all arguments to the `gcc` command. (Technically `gcc` is an argument as well, as we shall see.)

Command-line arguments are very useful. After all, C functions wouldn't be very useful if you couldn't ever pass arguments to them -- adding the ability to pass arguments to programs makes them that much more useful. In fact, all the arguments you pass on the command line end up as arguments to the `main` function in your program.

Up until now, the skeletons we have used for our C programs have looked something like this:  

```
#include <stdio.h>

int main()
{

  return 0;
}
```

From now on, our examples may look a bit more like this:  

```
#include <stdio.h>

int main (int argc, char \*argv\[\])
{

  return 0;
}
```

As you can see, `main` now has arguments. The name of the variable `argc` stands for "argument count"; `argc` contains the number of arguments passed to the program. The name of the variable `argv` stands for "argument vector". A vector is a one-dimensional array, and `argv` is a one-dimensional array of strings. Each string is one of the arguments that was passed to the program.

For example, the command line  

```
gcc -o myprog myprog.c
```

would result in the following values internal to GCC:

  

`argc`

4  

`argv[0]`

`gcc`  

`argv[1]`

`-o`  

`argv[2]`

`myprog`  

`argv[3]`

`myprog.c`

As you can see, the first argument (`argv[0]`) is the name by which the program was called, in this case `gcc`. Thus, there will always be at least one argument to a program, and `argc` will always be at least 1.

The following program accepts any number of command-line arguments and prints them out:  

```
#include <stdio.h>

int main (int argc, char \*argv\[\])
{
  int count;

  printf ("This program was called with \\"%s\\".\\n",argv\[0\]);

  if (argc > 1)
    {
      for (count = 1; count < argc; count++)
	{
	  printf("argv\[%d\] = %s\\n", count, argv\[count\]);
	}
    }
  else
    {
      printf("The command had no other arguments.\\n");
    }

  return 0;
}
```

If you name your executable `fubar`, and call it with the command `./fubar a b c`, it will print out the following text:  

```
This program was called with "./fubar".
argv\[1\] = a
argv\[2\] = b
argv\[3\] = c
```