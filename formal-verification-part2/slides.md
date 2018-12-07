---
theme : "league"
transition: "slide"
highlightTheme: "none"
---

## Formal Verification
## (Part 2)

<small>

Dominik Teiml

dominik@gnosis.pm

dteiml.github.io/presentations --> Formal Verification Part 2

</small>

<!-- --- -->

<!-- <img data-src="./graph.png" style="background: none !important; border: none !important;">

-- -->

<!-- <img data-src="./graph.png" style="background: none !important; border: none !important;"> -->

---

<!-- TODO: Diagram -->
<!-- TODO: Recap of Part 1 -->

## Symbolic execution

--

- A program can be executed *concretely* (with specific arguments)
- Or *symbolically* - arguments are treated as symbols (variables)
- We go through the program
  <!-- control-flow graph of the pgm -->
- Branching can occur e.g. at conditional statements like if, while, ...

--

- Can also occur in normal execution
    - e.g. consider the EVM opcode ADD
  <!-- (or equivalently, the high-level Solidity operator `+`) -->
    - If both arguments $A, B$ are unconstrained EVM words ($[0, 2^{256})$), the image can either be $A+B$ or $A+B-2^{256}$.
    - For EXP there will be an enormous amount of possibilities
<!-- (this is called path explosion) -->

--

- A "path condition" are the constraints that make the path "feasible" (reachable)
- Updating the path condition is equivalent to branching
- A solution to the path condition is a test input
  <!-- for that path -->

--

So K framework uses symbolic execution to prove program correctness. But how does it actually prove it?

---

## Z3

--

<!-- Let's take a step back -->

SAT

- Problem of the satisfiability of boolean formulas
- E.g. does there exist an *interpretation* for $A,B,C$ such that $\lnot A \lor (B \land C)$?
- Known to be NP-Complete by Cook-Levin Theorem

--

<!-- - If we want to find a solution to a SAT problem, we use a SAT solver -->
SAT Solvers
<!-- which run in hyperpolynomial time -->
- Takes as input a CNF boolean expression and outputs either a solution or UNSAT
- We use CNF because it better corresponds to most use cases (e.g. path conditions :))
- And because there is a polynomial-time reduction from SAT to 3-SAT (cf. Karp)
- (NB DNF can be solved in polynomial time)
<!-- but no algorithm exists to transform an arbitrary boolean formula to DNF) -->
<!-- In most of the above, finding such an algorithm would imply P=NP, so it's very likely we'll never find it :( -->

--

<!-- - For formal verification we need more expressiveness than just boolean formulas -->
SMT Solvers

- Satisfiability *modulo theories* 
<!-- Theory here has its mathematical meaning -> axiomatic system -->
- E.g. integer arithmetic
<!-- Again -->
- The solver will either output a solution or UNSAT

--

<!-- So how is this useful for formal verification? -->
- How do we *prove* positive properties (over a set of integers)?
    - e.g. let $x$ be even. Prove $x^2 \equiv 0 \mod 4$
- We can feed the variable ($x$), constraint (parity) and the equation ($x^2 \not\equiv 0 \mod 4$) 
  
  into an SMT-Solver 

- If it outputs UNSAT, we proved the property :P

--

Z3 is an instantiation of an SMT-solver
<!-- In practice, SMT-Solvers first try to solve the SAT problem and if that is satisfiable then they see if the constraints posed by the propositions are consistent -->

---

## Intermezzo
--

High-level overview:
- K executes your program symbolically, branching if necessary
- At some point, Z3 will try to prove the post-condition
- If it can do so for all paths, the spec is proved
- If at least one path has an irreducible term, it terminates and spec is not proved
- You start by specifying the pre-condition: initial contents of your language configuration
    - We can use various things:
    - TODO:
    - variables and their constraints (e.g. $0 < SENDER_ADDRESS < 2^160$)


Formal verification = formal specification (manual) + formal proof (automated)

---

## K Specifications
<!-- In this section we'll connect what we've discussed to how to actually use K -->

--

A K specification usually involves exactly one *spec rule*
- Unlike *semantic rules* which specify rewrite rules, it specifies what we want to prove
- In particular, a pre-condition 
    - initial contents of configuration cells
    <!-- Which depends on the language e.g. EVM -->
- and post-condition
    - desired contents
    <!-- of configuration cells after execution -->
- which are separated by a `=>`
- Together, the pre- and post-conditions (when used for program correctness) express the intended behavior of the contract

--

<!-- So a K specification must contain pre- and post-conditions separated by => -->

What are the different possibilities in one cell?

<!-- The most popular are -->

--

No rewrite

<small>

| Syntax         | Examples                                                                                                         |
| -------------- | ---------------------------------------------------------------------------------------------------------------- |
| Value type     | `<callValue> 0 </callValue>`, `<mode> BYZANTIUM </mode>`, `<code> #parseByteStack("...") </code>`                |
| Anonymous var  | `<callStack> _ </callStack>`, `<gasPrice> _ </gasPrice>`, `<blockhash> _ </blockhash>`, `<balance> _ </balance>` |
| Named variable | `<callValue> CALL_VALUE </callValue>`, `<caller_id> MSG_SENDER </caller_id>`                                     |

</small>

--

With rewrite sign

<small>

|   Examples    |
|  ---  |
|  `<k> #execute => #halt </k>`,      |
|  `<output> _ => #asByteStackInWidth(1, 32) </output>`     |
|  `<pc> 0 => _ </pc>` <br> `<wordStack> .WordStack => _ </wordStack>` <br> `<gas> 100000 => _ </gas>`    |


</small>

```
#hashedLocation("Solidity", 0, CALLER_ID) |-> (BAL_FROM => BAL_FROM -Int VALUE)
#hashedLocation("Solidity", 0, TO_ID)     |-> (BAL_TO   => BAL_TO   +Int VALUE)
_:Map
```








--

<!-- Let's look at an example -->

<!-- Let's say we have a super simple imperative language IMP that has a configuration -->

```
configuration <T>
                <k> $PGM:Pgm </k>
                <state> .Map </state>
              </T>
```

Then a possible K specification would be

```
rule <T>
        <k>Y = X; => . </k>
        <state> X |-> X_VALUE  _:Map => Y |-> X_VALUE _:Map </state>
     </T>
```

--

We can also include constraints on our variables.

```
rule <T>
        <k> Y = X % 2; => . </k>
        <state> X |-> X_VALUE _:Map => Y = 0 _:Map </state>
     </T>
    requires #even(X_VALUE)
```

<!-- And then twe would have another one for odd X -->

<!-- Ok, that was easy, let's try  -->

<!-- TODO: Different ways to write

e.g. -->

--

```
requires "abstract-semantics.k"
requires "verification.k"

module TRANSFER-SUCCESS-1-SPEC
  imports ETHEREUM-SIMULATION
  imports ABSTRACT-SEMANTICS
  imports VERIFICATION

  // transfer-success-1
  rule
    <k> #execute => #halt </k>
    <exit-code> 1 </exit-code>
    <mode> NORMAL </mode>
    <schedule> BYZANTIUM </schedule>
    <analysis> .Map </analysis> // not part of evm

    <ethereum>
      <evm>
        <output> _ => #asByteStackInWidth(1, 32) </output>
        <statusCode> _ => EVMC_SUCCESS </statusCode>
        <callStack> _ </callStack>
        <interimStates> _ </interimStates>
        <touchedAccounts> _ => _ </touchedAccounts>

        <callState>
          <program> #asMapOpCodes(#dasmOpCodes(#parseByteStack("..."), BYZANTIUM)) </program>
          <programBytes> #parseByteStack("...") </programBytes>
          <id> ACCT_ID </id> // contract owner
          <caller> CALLER_ID </caller> // who called this contract; in the begining, origin // msg.sender

          <callData> #abiCallData("transfer", #address(TO_ID), #uint256(VALUE)) </callData>
          <callValue> 0 </callValue>
          <wordStack> .WordStack => _ </wordStack>
          <localMem> .Map => _ </localMem>
          <pc> 0 => _ </pc>
          <gas> 100000 => _ </gas>
          <memoryUsed> 0 => _ </memoryUsed>
          <previousGas> _ => _ </previousGas>

          <static> false </static> // NOTE non-static call
          <callDepth> CALL_DEPTH </callDepth>
        </callState>

        <substate>
          <selfDestruct> _ </selfDestruct>
          <log> _:List ( .List => ListItem(#abiEventLog(ACCT_ID, "Transfer", #indexed(#address(CALLER_ID)), #indexed(#address(TO_ID)), #uint256(VALUE))) ) </log>
          <refund> _ => _ </refund> // TODO: more detail
        </substate>

        <gasPrice> _ </gasPrice>
        <origin> ORIGIN_ID </origin> // who fires tx

        <previousHash> _ </previousHash>
        <ommersHash> _ </ommersHash>
        <coinbase> _ </coinbase>
        <stateRoot> _ </stateRoot>
        <transactionsRoot> _ </transactionsRoot>
        <receiptsRoot> _ </receiptsRoot>
        <logsBloom> _ </logsBloom>
        <difficulty> _ </difficulty>
        <number> _ </number>
        <gasLimit> _ </gasLimit>
        <gasUsed> _ </gasUsed>
        <timestamp> _ </timestamp>
        <extraData> _ </extraData>
        <mixHash> _ </mixHash>
        <blockNonce> _ </blockNonce>

        <ommerBlockHeaders> _ </ommerBlockHeaders>
        <blockhash> _ </blockhash>
      </evm>

      <network>
        <activeAccounts> SetItem(ACCT_ID) _:Set </activeAccounts>

        <accounts>
          <account>
            <acctID> ACCT_ID </acctID>
            <balance> _ </balance>
            <code> #parseByteStack("...") </code>
            <storage>
              #hashedLocation("Solidity", 0, CALLER_ID) |-> (BAL_FROM => BAL_FROM -Int VALUE)
              #hashedLocation("Solidity", 0, TO_ID)     |-> (BAL_TO   => BAL_TO   +Int VALUE)
              _:Map
            </storage>
            <origStorage> _ </origStorage>
            <nonce> _ </nonce>
          </account>
          ...
        </accounts>

        <txOrder> _ </txOrder>
        <txPending> _ </txPending>
        <messages> _ </messages>
      </network>
    </ethereum>
    requires 0 <=Int ACCT_ID    andBool ACCT_ID    <Int (2 ^Int 160)
    andBool 0 <=Int CALLER_ID   andBool CALLER_ID  <Int (2 ^Int 160)
    andBool 0 <=Int ORIGIN_ID   andBool ORIGIN_ID  <Int (2 ^Int 160)
    andBool 0 <=Int CALL_DEPTH  andBool CALL_DEPTH <Int 1024
    andBool 0 <=Int TO_ID       andBool TO_ID     <Int (2 ^Int 160)
    andBool 0 <=Int VALUE       andBool VALUE     <Int (2 ^Int 256)
    andBool 0 <=Int BAL_FROM    andBool BAL_FROM  <Int (2 ^Int 256)
    andBool 0 <=Int BAL_TO      andBool BAL_TO    <Int (2 ^Int 256)
    andBool CALLER_ID =/=Int TO_ID
    andBool VALUE <=Int BAL_FROM
    andBool BAL_TO +Int VALUE <Int (2 ^Int 256)

endmodule
```

<!-- -- -->

<!-- TODO: eDSL, spec templates -->

<!-- TODO: DEMO! -->



<!-- Execution of a path ends when when it either:

Matches the implication
No other spec/regular rule can match.
<k> cell matches the <k> cell of the target term (but only for cases when initial term was not matching the target <k>). This is a customization in gnosis branch of K, to avoid further evaluation when it is clear that this path cannot be proved.
In the 2nd and 3rd case specification failed for this particular path.

--

When evaluation of all paths finishes, kprove will output either #True if proof succeeded, or the list of paths that failed. Paths on which proof succeeded won't be displayed. -->

<!-- -- -->



---

<!-- .slide: style="text-align: left;" -->
## THE END