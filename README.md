##Extending IBM i RPG assets with ASNA Visual RPG

*All* IBM i shops have legacy RPG applications. These applications are typically built out of hundreds, sometimes thousands, of RPG programs. It's not uncommon for many of those RPG programs to be reusable program objects that are called by many other RPG programs in the application. 

Extending the life and value of reusable RPG program projects is something that [ASNA Visual RPG](https://asna.com/us/products/visual-rpg) (AVR) is very good at. You can build a fat Windows client, or an ASP.NET Website, with ASNA Visual RPG and then easily reuse your existing RPG program object portfolio in these apps. Although this article shows AVR calling an ILE RPG-generated program object, AVR can call an IBM program object generated from any language.    

This product reviews shows you how simple it is to use AVR to reuse your existing IBM i program objects.

#### Calling IBM i program objects 

AVR supports callings IBM i program objects with the familiar RPG Call/Parm programming interface. Like RPG on the IBM i, AVR's program call obeys the pass-by-reference semantics for program call parameters. When the IBM i program object changes a parameter value, that change is recognized in AVR after the call. 

In addition to the typical scalar parameter types (packed, zoned, char, etc), AVR can also pass data structures and multiple-occurrence data structures with its program call. Let's take a look at an example. 

Figure 1a below is the ILE RPG program we'll call with AVR. It is written using [IBM i's TR7 free-format syntax](https://www.ibm.com/developerworks/ibmi/library/i-ibmi-rpg-support/). Almost. I couldn't for the life of me get a PList to work with TR7 ILE RPG so I had to jam in two fixed-format lines. If you can TR7-ize lines 8 and 9 for me I’ll buy you a cold one!  

This program populates an externally-defined data structure array with two fields from a file. That data structure is the single parameter for the program call.  
          
    0001  Ctl-Opt Option(*srcstmt) Dftactgrp(*No) ActGrp('rptest');
    0002  
    0003  Dcl-F CustomerL2 Disk(*ext) Usage(*Input) Keyed;
    0004  
    0005  Dcl-DS CustDS LikeRec(RCMMASTER:*Key) Dim(200);
    0006  
    0007  /end-free
    0008  c     *entry        plist
    0009  c                   parm                    CustDS
    0010  /free
    0011  
    0012  LoadCustDS();
    0013  
    0014  *InLR = *On;
    0015  Return;
    0016  
    0017  Dcl-Proc LoadCustDS;
    0018      Dcl-S RowCount Packed(12:0);
    0019  
    0020      RowCount = 0;
    0021  
    0022      DoW (RowCount < 16);
    0023          RowCount = RowCount + 1;
    0024          Read CustomerL2;
    0025          CustDS(RowCount).CMCustNo = CMCustNo;
    0026          CustDS(RowCount).CMName = CMName;
    0027      EndDo;
    0028  End-Proc;

###### Figure 1a. ILE RPG program to be called by AVR.

The ASNA Visual RPG code to call Figure 1a's ILE RPG program is shown below. A small code narrative follows this code listing.  

    0001  DclDB pgmDB DBName("*PUBLIC/Cypress")
    0002  
    0003  DclDs CustDS Dim(200)   
    0004  DclDsFld CMName   Type(*Char) Len(40) 
    0005  DclDsFld CMCustNo Type(*Packed) Len(9,0)
    0006  
    0007  BegFunc ProgramCallWithDS Access(*Public) Type(*Integer4) 
    0008      DclFld Counter Type(*Integer4) 
    0009  
    0010      Call "*Libl/PgmCall" DB(pgmDB) 
    0011      DclParm CustDS 
    0012  
    0013      Do FromVal(1) ToVal(200) Index(Counter) 
    0014          Occur CustDS NewIndex(Counter) 
    0015  
    0016          If (CMCustNo = 0)
    0017              Leave
    0018          EndIf          
    0019      EndDo 
    0020  
    0021      LeaveSr Counter - 1
    0022  EndFunc 

###### Figure 1b. AVR snippet to call Figure 1a's ILE RPG program object.

**Line 1.** 

AVR's database access and program calls are provided by [ASNA's DataGate](https://asna.com/us/products/datagate). DataGate is a TCP/IP-based IBM i host server that connects a Windows PC or server to an IBM i. DataGate is secure, performant, and very easy to configure. 

AVR provides a superset of RPG operation codes. Its `DclDB` operation code defines the active `database name` for this program. In AVR parlance, a database name identifies a centrally located set of database connection properties (ie, IBM i IP address, user profile name, IBM i password, connection pool time, etc). This database name provides AVR with the information it needs to pass along to DataGate to perform file IO and program calls.

**Lines 3-5.**   

These three lines define a data structure array. These lines provide a data structure identical to the one defined in Figure 1a's line 5. TR7 ILE RPG added the handy *KEY keyword causes an external data structure to include only the key fields; AVR's structure needs to explicitly defined. 

**Lines 7.**

AVR supports both subroutines and functions. Variables defined in these routines are local to the routine. AVR's subroutines and functions can also have parameters passed to them. They are essentially AVR's streamlined answer to ILE RPG's subprocedures.   

**Line 8.** 

Declare local variable `counter` to track how many elements are written to the data structure array.

**Lines 10-11.**

These two lines perform the program call to the IBM i. The `DB` keyword identifies the IBM i (through its database name) to which this program call is occurring. You can fully quality the program object being called or use, as this example shows, the `*Libl` library list keyword.

**Lines 13-19.**

The ILE RPG program called arbitrarily defined 200 elements in its data structure array (line 5 of Figure 1a). Lines 13-19 loop over the data structure array elements passed to AVR from the IBM i to count them. 

**Lines 21.**

AVR's arrays are zero-based (ie, the first element in an AVR array is the zeroth element). To return the number of elements populated by the RPG program, the `counter` value minus 1 is returned.

To benchmark this program call, I modified both programs slightly to conditionally end the RPG program call. When calling the open RPG program, call times averaged about 140 milliseconds per call.   
 
#### Extending IBM i RPG assets

As you can see, it is very easy to make program calls to IBM i program objects from ASNA Visual RPG. These calls work from either fat Windows AVR programs, or AVR-powered Web sites or Web services. In any use case, AVR's program calls are one of the big ways that AVR can let you extend the life and value out of your IBM i RPG assets. AVR gives those old program objects a lease on life.      
 
The full code for this example, including the ILE RPG program, are online at ASNA's [GitHub account](https://github.com/ASNA/AVR-Program-Call).   

####A parting shot

I've seen articles recently bemoaning the slow take-up of TR7's vastly improved free-format syntax. Given the sorry state of TR7-specific free-format docs, it's no wonder no one is using it. You quite nearly need to have  [Kreskin](http://www.amazingkreskin.com/) on your programming team fully exploit this great new syntax. Both Jon Paris and Susan Ganter's [IBM Systems Magazine blog](http://www.ibmsystemsmag.com/Blogs/iDevelop/) and the occasional Barbara Morris [PowerPoint](http://www.ocean400.org/assets/whats_new_in_rpg_for_7.2.pdf) are better help than anything you'll find in the IBM i white books. Also take a look at this [simple TR7 ILE RPG example](https://asna.com/us/articles/newsletter/2015/q2/ile-rpg-goes-free). 

I'm nearly certain that the back of a cereal box would be more helpful than any of IBM i's RPG documentation! Isn't it amazing how some things never change? My documentation beef aside, IBM i's TR7 ILE RPG makeover is very highly recommended. It is expressive, mostly predictable, and vastly smooths out the syntactic thorns that have dogged ILE RPG since the mid 90s. 
