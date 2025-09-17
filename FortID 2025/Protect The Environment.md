## The challenge:
We get a binary which gives us the option to either print an environment variable, or "protect" an environment varialble using rot13. This will get a pointer to the environment variable in libc's stack frame, and alter it in place. Memory-wise, the rot13 function is airtight:
```c
void rot13(char *s) {
  while (*s != 0) {
    *s += 13;
    s++;
  }
}
```

The flag is stored in the `FLAG` environment variable, and we have the following check on our input when we print an environment variable:
```c
if (!strcmp(name, "FLAG")) {
        printf("Access denied\n");
} 
```

## The (dumb) solution
So, the first path that I went down after realizing that any kind of memory error (overflows, format string exploits, and freelist shenanigans) was out of the question, was investigating how environment variables were processed, how our string comparison is processed, and what edge cases we could find.

First, let's take a look at *exactly* what our string comparison function is doing, given the several similar functions in C. The `strcmp` function compares two strings up until the null terminator is encountered. This means that unlike `strstr`, another commonly used function for this purpose, we can add something onto the end and get the comparison to return false.
For example:

```c
strcmp("FLAGabc", "FLAG") != 0; //Comparison returns 0 when strings match

strstr("FLAGabc", "FLAG") = 0; //Comparison still matches
```

Next, let's look at how the environment variables are stored in memory. On the stack, libc creates a list of pointers to strings, with one string per env variable. Notably, the key-value pairs are stored as a single string. So for example, one entry might consist of a pointer to the string `FLAG=thisIsTheFlag`.

So, I figured that if we ran the `rot13` function on the FLAG env variable until it contained a second equals sign, then it would be parsed weird, and the environment variable would be `FLAG=`, some junk, let's say `:3` for example, a second equal sign, and some more junk composed of the rotated flag.
I could then get the flag by printing the environment variable called `FLAG=:3`, seperated from the rest of the rotated flag by the second `=` character we rotated into, and then rotate everything back for the flag. 

So, I implemented all that, got the flag, and felt good about myself, and then my teammate asked me to do some pwn writeups. So I thought it would be cool to go into the libc source code to explain why this parsing quirk worked. And that brings us to..

## The (not dumb) solution



So, let's take a look at the internal function called by `getenv()` inside of libc, which I've tidied up and commented:

```c
char * __findenv(const char *name)
{
  //Our array of string pointers as mentioned earlier
	extern char **environ;

	int len, i;
	const char *namePtr;
	char **p, *envVarPtr;
	if (name == NULL || environ == NULL)
		return (NULL);

  //Now, we get the length of our name pointer. Notice that the delimiter is the = character.
	for (namePtr = name; *namePtr != '='; ++namePtr)
		;
 
	len = namePtr - name;

  //Iterate over our environment variables
	for (p = environ; (envVarPtr = *p) != NULL; ++p) {

		for (namePtr = name, i = len; i && *envVarPtr; i--)
      //If we haven't gotten to the delimiter yet and they don't match, this isn't the right environment variable
      if (*envVarPtr++ != *namePtr++)
				break;
       
		if (i == 0 && *envVarPtr++ == '=') {
			return (envVarPtr);
    }
  }
	return (NULL);
}
```

If we look at the function, it only compares our environment variable's name up to the `=` character. So, if we had just done `print FLAG=`, we would have passed the string comparison check and been able to retrieve the right environment variable without ever having to use the rot13 command. 

## It feels weird to just cut it off right here so here's a couple more sentences
So, if there's a good thing to remember from this challenge aside from cool libc internals, it's that reading libc source code is much more useful and much less scary than I (and maybe you) thought.

i guess you're supposed to plug your stuff at the end of writeups, right? my instagram is @e7han_b, i play drums and stuff but i dont have anything else CTF-related

