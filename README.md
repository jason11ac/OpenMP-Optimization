# OpenMP-Optimization
Using OpenMP to optimize a program in C

~~~~~
Goal
~~~~~
To use OpenMP ( http://openmp.org/wp/ ) to make a program run faster. 

~~~~~~~~~~~~~~~
OpenMP.txt
~~~~~~~~~~~~~~~

~~~~~~
SETUP
~~~~~~

Throughout the lab I referenced my TA's slides for help on how to set up the lab and what to
experiment with. 


To setup the files, I made openmplab.tar into a zip file and scp'd that to my linux server using:

	- scp openmplab.zip alvarezc@lnxsrv09.seas.ucla.edu:~/

Then I unziped it using:
	
	- unzip openmplab.zip -d ~

Next I made seq and then executed it. The output I received was two different times:

	FUNC TIME : 0.474964
	TOTAL TIME : 2.451661

Then I made omp and executed it and the times were around the same numbers because essentially 
both seq and omp are the same. The only difference between the two is that OpenMP is enabled 
with omp and not with seq, but because I haven't added anything to my omp it should be the same,
which it is.  

Now all we have to do is start speeding it up with OpenMP.


~~~~~~~~~~~
OpenMP Lab
~~~~~~~~~~~

The first thing I wanted to do was use GPROF to see which functions were taking most of the CPU and real time. 
Using GPROF with the less command:

	gprof omp | less

I got this initial table:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 63.72      0.35     0.35       80     4.38     4.92  func1
 21.85      0.47     0.12  5177344     0.00     0.00  func2
  3.64      0.49     0.02   491520     0.00     0.00  addSeed
  3.64      0.51     0.02       17     1.18     6.90  func4
  3.64      0.53     0.02                             rand2
  1.82      0.54     0.01        2     5.01    60.24  func5
  1.82      0.55     0.01                             filter
  0.00      0.55     0.00       16     0.00     0.00  func0
  0.00      0.55     0.00        1     0.00     0.00  func3

which tells me that func1 is taking most of the function time. Therefore this is function I should target
first in order to reduce the speed. After looking at func1 using vi, I decided to put the first loop in a 

	#pragma omp parallel for private (i)

because I wanted each for loop and each instance of i to be private and not shared by the threads. (synchronization) 
After adding this simple line, my function time went from ~0.474964 to ~0.373898. This is a little faster but
not fast enough. 

I experimented with a couple more lines including putting brackets with my pragma lines which gave me segmentation
faults and slower results. I then used the same method as I did with the first for loop and tried it on the
second, adding the above line before the other loops as well. 

My output was:
	
	FUNC TIME : 1.372607
	TOTAL TIME : 3.379219 

This is substantially slower than what I started with. That's not good. I then decided to include all
the variables the loop uses in private because each thread should have it's own instance of each variable
in order for the loop to have defined behavior. (synchronization) So I changed the above line to:

	#pragma omp parallel for private (i, j, index_X, index_Y)

and inserted that before the second loop of func1. With only these two added lines my output was:

	FUNC TIME : 0.094115
	TOTAL TIME : 2.064764

Wow. I was really surprised by how much of a difference this line made. According to the function time
I had successfully sped my time up by a factor of 0.474964 / 0.094115 = 5.04663444. This is way better
than the 3.5 I was expected to reach. My new gprof table was:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 75.54      0.43     0.43                             filter
 19.32      0.54     0.11  4264113     0.00     0.00  func4
  1.76      0.55     0.01        1    10.01   106.31  rand2
  1.76      0.56     0.01                             addSeed
  1.76      0.57     0.01                             fillMatrix
  0.00      0.57     0.00   491520     0.00     0.00  dilateMatrix
  0.00      0.57     0.00       45     0.00     0.00  func2
  0.00      0.57     0.00       18     0.00     6.02  func5
  0.00      0.57     0.00       16     0.00     0.05  func0
  0.00      0.57     0.00       15     0.00     0.00  func1
  0.00      0.57     0.00        1     0.00     0.00  init

As you can see, the func1 now takes 0.00 percent of the function time which is perfect. 

Even though I was done with my assignment I wanted to see if I could speed any other parts of the entire 
function up even more. Looking at the original gprof table:


Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 63.72      0.35     0.35       80     4.38     4.92  func1
 21.85      0.47     0.12  5177344     0.00     0.00  func2
  3.64      0.49     0.02   491520     0.00     0.00  addSeed
  3.64      0.51     0.02       17     1.18     6.90  func4
  3.64      0.53     0.02                             rand2
  1.82      0.54     0.01        2     5.01    60.24  func5
  1.82      0.55     0.01                             filter
  0.00      0.55     0.00       16     0.00     0.00  func0
  0.00      0.55     0.00        1     0.00     0.00  func3

we can see that besides func1, which is now taking 0.0 % of the time, func2, addSeed and func4 also take 
some time. So I then began to see if I could speed those up. Using the same private method as above, I 
added the line:
	
	#pragma omp parallel for private (i)

before the three loops in func2 because each one uses and increments it's own i variable. 
My output speeds now read:

	FUNC TIME : 0.079331
	TOTAL TIME : 2.094355

which is even faster than my last test of 0.094115. I now had a function speedup factor of 0.474964 / 0.079331 = 5.98711727.
A speedup factor of roughly 6 was great as our original requirement was 3.5 as stated above. I then went to the util.c 
file that contained the addSeed function. In this function, three variables x, y, and z were being used by one loop
function so I added the line:

	#pragma omp parallel for private (x, y, z)

before the only loop. My output was now:

	FUNC TIME : 0.083330
	TOTAL TIME : 1.762039 

This shows us just a decrease in total time by about .3. Which is not bad, but our function time remains the same. 
After making this change to util.c I noticed that when inputting the make clean command I got an error. 
So I deleted this change in util.c and the error went away.

I then tried changing func4. func4 only contains one loop as well so I added the line:

	#pragma omp parallel for private (i)

as the loop only declared and used the variable i. My output was now:

	FUNC TIME : 0.079016
	TOTAL TIME : 1.749387

showing no significant change. My gprof table now showed:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 75.54      0.43     0.43                             filter
 10.54      0.49     0.06                             func1
  8.78      0.54     0.05  4264111     0.00     0.00  func0
  1.76      0.55     0.01        3     3.34     3.34  func5
  1.76      0.56     0.01                             rand1
  1.76      0.57     0.01                             rand2
  0.00      0.57     0.00   491520     0.00     0.00  fillMatrix
  0.00      0.57     0.00        1     0.00     0.00  func2

Looking at the above gprof table, I observed that now func0 was taking quite
a percent of the time. I then went into func.c to func0 and added the line:

	#pragma omp parallel for private (i)

around the single for loop in func0. My output then read:

	FUNC TIME : 0.077422
	TOTAL TIME : 1.859590

And my gprof table read:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 94.86      0.54     0.54  4194304     0.00     0.00  filter
  3.51      0.56     0.02        1    20.03    20.03  func5
  1.76      0.57     0.01                             rand2
  0.00      0.57     0.00   491520     0.00     0.00  addSeed
  0.00      0.57     0.00        1     0.00     0.00  func1
  0.00      0.57     0.00        1     0.00     0.00  func3

Using the private variable method I added a private variable line
before every for loop in every function in func.c. This can 
all be seen in my func.c file. Once adding the pragma line before
all the function loops, my make check command once again gave me an 
error. After experimenting with eliminating each one, one after the other, 
I realized that this was happening because there were too many 
pragma's in func2 and func3 could simply not be optimized using openmp. 
After eliminating 2 lines, one from func2 and one from func3, the error 
disappeared and my output and gprof table were as follows:

	FUNC TIME : 0.039544
	TOTAL TIME : 2.260421

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 94.86      0.54     0.54       17    31.81    31.81  filter
  1.76      0.55     0.01                             func2
  1.76      0.56     0.01                             rand1
  1.76      0.57     0.01                             rand2
  0.00      0.57     0.00   491520     0.00     0.00  addSeed
  0.00      0.57     0.00        1     0.00     0.00  func1
  0.00      0.57     0.00        1     0.00     0.00  func3

This was fantastic as my function time was now being consistently sped up 
by a factor of around 0.474964 / 0.039544 = 12.0110257! 

Let us compare the two executable seq and omp:

seq:

	FUNC TIME : 0.474964
	TOTAL TIME : 2.451661

omp:

	FUNC TIME : 0.039544
	TOTAL TIME : 2.260421

It is clear that the function time has been significantly optimized. 
After finishing my optimizations I made sure to run all the necessary tests 
to make sure everything was correctly done. 

1. make check:

	gcc -o omp  -O3 -fopenmp filter.c main.c func.c util.c -lm
	cp omp filter
	./filter
	FUNC TIME : 0.040323
	TOTAL TIME : 2.446596
	diff --brief correct.txt output.txt

	Because it returned silently with no errors, this check passed. 

2. make checkmem:

	Running this command gave me a bunch of addresses, which according to the
	TA is not a problem:

Memory not freed:
-----------------
           Address     Size     Caller
0x00000000016d16b0   0x1c40  at 0x7f9945c7b7b9
0x0000000000ceccc0     0xc0  at 0x7f9945c7b7b9
0x0000000000cecd90    0x108  at 0x7f9945c7b809
0x0000000000cecea0    0x240  at 0x7f994619fde5
0x0000000000ced0f0    0x240  at 0x7f994619fde5
0x0000000000ced340    0x240  at 0x7f994619fde5
0x0000000000ced590    0x240  at 0x7f994619fde5
0x0000000000ced7e0    0x240  at 0x7f994619fde5
0x0000000000ceda30    0x240  at 0x7f994619fde5
0x0000000000cedc80    0x240  at 0x7f994619fde5
0x0000000000ceded0    0x240  at 0x7f994619fde5
0x0000000000cee120    0x240  at 0x7f994619fde5
0x0000000000cee370    0x240  at 0x7f994619fde5
0x0000000000cee5c0    0x240  at 0x7f994619fde5
0x0000000000cee810    0x240  at 0x7f994619fde5
0x0000000000ceea60    0x240  at 0x7f994619fde5
0x0000000000ceecb0    0x240  at 0x7f994619fde5
0x0000000000ceef00    0x240  at 0x7f994619fde5
0x0000000000cef150    0x240  at 0x7f994619fde5
0x0000000000cef3a0    0x240  at 0x7f994619fde5
0x0000000000cef5f0    0x240  at 0x7f994619fde5
0x0000000000cef840    0x240  at 0x7f994619fde5
0x0000000000cefa90    0x240  at 0x7f994619fde5
0x0000000000cefce0    0x240  at 0x7f994619fde5
0x0000000000ceff30    0x240  at 0x7f994619fde5
0x0000000000cf0180    0x240  at 0x7f994619fde5
0x0000000000cf03d0    0x240  at 0x7f994619fde5
0x0000000000cf0620    0x240  at 0x7f994619fde5
0x0000000000cf0870    0x240  at 0x7f994619fde5
0x0000000000cf0ac0    0x240  at 0x7f994619fde5
0x0000000000cf0d10    0x240  at 0x7f994619fde5
0x0000000000cf0f60    0x240  at 0x7f994619fde5
0x0000000000cf11b0    0x240  at 0x7f994619fde5
0x0000000000cf1400    0x240  at 0x7f994619fde5

