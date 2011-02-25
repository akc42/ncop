Nested Closure Oriented Progamming
=================================
Alan Chandler <alan@chandlerfamily.org.uk>
v0.1, 25th February 2011

This document describes the rationale for the development of my Nested
Closure Oriented Programing framework - a Mootools Javascript Class
which can be used to support long running Javascript applications

== Introduction

If you have a long running Javascript program, most browsers will
freeze during its operation and, if it runs for a particularly long
time, will prompt the user to kill it. 

There is a technique to restructure such programs using a nested
closures, such that variables can be retained over a period of steps,
but that between each step the program returns to the browser to let
it deal with other activies.  I first encountered the techniques when
looking at some software to calculate a RSA public/private assymetic
key pair for use in a secure version of my chat program.  Even when
calculating a relatively small (64bit) key it takes over a second of
CPU time to calculate the key pair.  This technique prevents any
issues - and it is even possible to run applications that run for
several hours.

The technique was proposed by Atsushi Oka (http://oka.nu) and he had
ported some crytographic routines to use his implementation of it.
However when I came to use it, I discovered he had 

a. Added functions to Object.Prototype which conflicted with mootools
(the javascript framework I was using), and

b. He seemed to have made it way more complex than necessary - so much
so that I failed to understand how it was working in key areas.

In the end I decided to Redo the technique by creating a simple
Mootools Javascript Class to handle it, and to port all of the
Javascript library necessary to calculate and RSA key pair to it.
This repository holds the results.  The main framework is in the top
level directory as file ns.js, the remainder is in the cipher
subdirectory.

== The Nested Closure Technique

As Atsushi Oka puts it, a normal technique probably would involve
somthing like the following:-

---------------------------------------------------
  for ( var i=0; i<10; i++ ) {
      for ( var j=0; j<10; j++ ) {
          for ( var k=0; k<10; k++ ) {
              trace( "i=" + i + " j=" + j + " k=" + k );
          }
      }
  }
---------------------------------------------------

this can be replaced via a series of function calls structured as
follows:-

---------------------------------------------------
    var i=0;
    return function() {
        if ( i++<10 ) {
            var j=0;
            return function() {
                if ( j++<10 ) {
                    var k=0;
                    return function() {
                        if ( k++<10 ) {
                            trace( "i="+i + " j="+j + " k="+ k );
                            return true;
                        } else {
                            return false;
                        }
                    }
                } else {
                    return false;
                }
            }
        } else {
            return false;
        }
    };
 ---------------------------------------------------

As you can see, the structure looks similar, and variables i, j and k
have similar scopes, but although the first one will complete its
execution before returning, the second one only executes one step, and
needs to called again repeatedly until it returns false.  That is
essence is the Nested Closure Technique.

== How the NS class works

NS is implemented to work with the Mootools framework.  This version
works with Mootools version 1.2. Some changes will have to be made for
it to work with Mootools 1.3.

Some key concepts are as follows

A context:: A context is a execution environment for a small portion of the
    overall task. It consists of a series of steps which undertaken to
    complete either the complete task or some subsection of it.
A step:: In its basic form, a step consists of a function which can perform
    a small portion of the overall task, either completely within
    itself - or by setting up the javascript variables to form a
    environment for a new context which then spawned to break the step
    into smaller substeps. 
A Do loop :: A series of Steps which are executed in a loop. Generally a new
    context is created in which to run the loop
A Step completion indicator::   A way of indicating that instead of just performing the next step
    (or returning to the beginning in a Do loop if it was the last
    step) an alternative. Three exist:-

   1. Continue - just repeats the step
   2. Again - starts the Do loop from the beginning
   3. Done - exit the Do loop and pop to the next level

A result:: A special object that may be used to propagate a result 

The class constructor takes two parameters.  The first is the context
of this task - the function that will be executed when the task is
started,  The second is a Boolean defining if this context is unique
in the overall environment of the web page.  If it is, then some of
the key functions are bound to the Window object, making the calling
syntax slightly simpler.  For instance we can finish the execution of
a loop with a call like this

-------------
return DONE();
-------------

rather than the slightly less natural

------------------
return this.DONE();
------------------


Once the class has been constructed, execution is started by a call to
the EXEC method of the class. This takes two parameters

1. An array of parameters to hand over to the initial context (ie the
function referenced in the class constructor)
2. A function to call when the execution completes.  Like all
functions it takes the result object as its only input parameter.

By way of illustration - this shows how the RSA object is started to
generate the asymetric key pair.

----------------------------------------------------------------
RSA.prototype.generateAsync = function(keylen,exp,result) {
 var self=this;
 var generator = new NS stepping_generate,true);
 var _result = function(keyPair) {
   result( self );
 };
 generator.EXEC([keylen,exp],_result);

...
----------------------------------------------------------------

It is called with keylen and exp parameters and has a callback result.

The main utility method is DO. This is called as part of the return
function of the constructor, but may also be called during the return
of a step function to initiate a lower level context. DO takes a
single parameter, an Array of steps to be executed in a loop. Each
step can be one of

* function - in which case it will an executed step 
* array - in which case there is an embedded DO assumed 
* EXIT - which is the number 0 bound to the window object meaning this
loop should exit at this point (normally this is the last element of a DO loop)

Originally other values (such as a string representing a jump label)
where allowed, but it turned out that the RSA key generation didn't
need them and so they were removed for simplicity.

A similar method called STEP. This just takes a single step
parameter(as described above).

To illustrate how these functions may be called this is a small
snippet from the stepping_generate function of RSA

---------------------------------------------------------
function stepping_generate (myArgs) {
    var B,E;
    B = myArgs[0];
    E = myArgs[1];
    var rng;

    var qs = self.splitBitLength( B );
    self._e(E);
    var ee = new BigInteger(self.e);
    var p1; 
    var q1; 
    var phi;
    return DO([

       // Step1.ver2
        function () {
            RSA.log("RSAEngine:1.1");
	    self.p = new
	    BigInteger();
	    rng = new SecureRandom();
	},
	function () {
	    RSA.log("RSAEngine:1.2");
	    return self.p.stepping_fromNumber1( qs[0], 1, rng );
        },
	[
	 // Step1.3 ver3
	    function () {
	        RSA.log("RSAEngine:1.3.1");
		if ( self.p.subtract(BigInteger.ONE).gcd(ee).compareTo(BigInteger.ONE)!= 0 ) return AGAIN();
	    },
         ....
----------------------------------------------------------------------------

The above illustration shows the use of the return AGAIN();
construct. This tells the engine to repeat that loop (which in the
above case is just that first step of the sub loop started with 
the array indicator just above the Step 1.3 comment).

There are three such methods, AGAIN(), CONTINUE() and DONE() - each of
which sets the step completion indicator. There methods also
optionally take a single parameter - which is the value of the result
object - its use is illustrated below

--------------------------------------------------------------------------------
    if ( x[0] == lowprimes[i] ) {
        BigInteger.log( "stepping_isProbablePrime.1 EXIT" );
	//return true;
	return DONE(true);
    }
    BigInteger.log("stepping_isProbablePrime.2 EXIT" );
    // return false;
    return DONE(false);
--------------------------------------------------------------------------------

and then a step which is using this result (its a layer above because
of the return from a DO loop signalled by the DONE() call ). The
result parameter is always available as the (only) parameter of a step
function.

-----------------------------------------------------------------------
function (result) {
	 RSA.log("RSAEngine:1.3.3 : returned stepping_isProbablePrime"
	 + result );
	 if ( result ) {
	    RSA.log("RSAEngine:1.3.3=>EXIT");
		return DONE();
		} else {
		  RSA.log("RSAEngine:1.3.3=>AGAIN");
			return AGAIN();
			}
},
-----------------------------------------------------------------------

Occassionally you may with to set the result but not immediately
return from the function. The SET method allows this
This is illustrated below - (the x.stepping_millerRabin(t) function
will change the result

----------------------------------------------------------------------
function(result) {
    BigInteger.log( "stepping_isProbablePrime No.2: calling millerRabin : subparam.result=" + result );
    SET(null);
    return x.stepping_millerRabin(t);
},
function(result) {
    BigInteger.log( "stepping_isProbablePrime No.3: returning millerRabin : subparam.result=" + result );
    BigInteger.log( "stepping_isProbablePrime No.3: param.result=" + result );
    return DONE();
}

----------------------------------------------------------------------

Internally the NS class operates in two phases. These are driven
around the call to DO or STEP. In the first phase these calls build a
Step based structure in a context and returns. In the second phase
these steps are executed step by step using a "ticker" function which
is called every tick and executes a single step, calling the function
embedded within the step definition. If one of the executing step
functions comes across a further embedded DO or STEP function, the
engine temporarily switches back to context construction mode and
builds the new steps into the structure and then returns, having set
up the ticker execution function to execute them when it is activated
on the next tick.

Behind the scenes this execution engine manages the contexts, and
passes the result object around.
