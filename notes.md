
# Secure SAT Solver

```cpp
// Created by Ning Luo on 5/18/21.
//

#include <math.h>
#include "formula.hpp"
#include <memory>
#include <chrono>
#include <exception>
#include <stdlib.h>
#include <ctime>
#include "clause.hpp"
#include "literal.hpp"
#include "model.hpp"
#include "utils.hpp"
#include "solver.hpp"
using namespace std;
using namespace emp;

int main(int argc, char** argv) {
    try {
        int port, party; // Party is ALICE or BOB
        //cout<<"finish" << endl;

        parse_party_and_port(argv, &party, &port);
        NetIO *io = new NetIO(party == ALICE ? nullptr : "127.0.0.1", port); // ALICE listens for a connection on the given port
```

**Explanation**:  

- In main, we first parse the command-line arguments to figure out which party we are (ALICE or BOB) and set up the network I/O.  
- NetIO *io = new NetIO: If this process is ALICE, it listens on the port. If it is BOB, it connects to the specified host/port.  

---

## Secure Computation Initialization

```cpp
        // Overall Flow of the Secure Computation Setup
        // Alice acts as generator and Bob acts as evaluator.
        // ALICE (Party 1)
        //     Uses HalfGateGen to generate encrypted circuit gates.
        //     Uses SemiHonestGen to execute the protocol.

        // BOB (Party 2)
        //     Uses HalfGateEva to evaluate the encrypted circuit.
        //     Uses SemiHonestEva to execute the protocol.
        // “We implemented our solver using the semi‐honest 2PC library of the EMP‐toolkit [60].”
        // allows the two parties to exchange garbled values during secure computations 

        setup_semi_honest(io, party);

        int number_of_steps = atoi(argv[3]);
        int nvar = atoi(argv[4]);
```

**Explanation**:  
- setup_semi_honest(io, party) use the semi-honest protocol from EMP (haven't looked too hard here).  
-  both parties follow the protocol but try not to reveal extra information. Alice is typically the “garbler”  and Bob is the “evaluator”  


---

## Constructing Secure Formulas and creating the Solver
### I believe this sections references this part of the paper 
```
"We begin by defining
a pair of abstract data structures for a formula φ and its con-
stituent clauses, and then describe an instantiation of these ob-
jects and their operations using bit-vectors"
```
## Understanding EMP bits
The "EMP Bit" that will be used quite a bit in this is from the EMP Toolkit where Each Bit represents a single boolean value (true or false) but in an encrypted or secret-shared form so that neither party sees the raw value, it supports logical ops,EMP handles all the encryption, garbling, and network communication needed to keep values hidden.
```cpp
        // Formula constructor

        // This Formula constructor I believe is explained by section 4.1 in the paper 
        // where it is explained further:
        Formula::Formula(int nvar, string text)
        {
            vector<string> raw_cls = Parser::parse_clauses(text);
            //active is just a vector of bits that is used to keep track of which clauses are active
            active = new Bit[raw_cls.size()];
            // "A formula φ is encoded as matrices (P, N) ... as well as a vector isAlive ∈ {0, 1}^m
            //  whose jth entry indicates whether Cj has been removed from the formula."
            // In our code, the active array plays the role of this “isAlive” vector.
            
            int i = 0;
            for (auto raw_cl : raw_cls) {
                active[i] = Bit(1, PUBLIC);
                as well as a vector
            // "isAlive ∈ {0, 1}m whose jth entry indicates whether Cj has
            // been removed from the formula. The φ.remove(C) function-
            // ality can be implemented by setting isAlive[ j] = 0"
               
                cls.push_back(make_unique<BIClause>(nvar, raw_cl));
                i++;
            }
        }

        auto phi_a = make_unique<Formula>(nvar, argv[5], ALICE);
        auto phi_b = make_unique<Formula>(nvar, argv[6], BOB);
```

**Explanation**:  
- `Formula` objects, `phi_a`  and `phi_b` 
- Each formula is built by parsing a text input (`argv[5]` for Alice’s, `argv[6]` for Bob’s), splitting it into clauses, and wrapping them in secure data structures (`BIClause`) with an associated `Bit` that indicates whether each clause is active.  
- `Seb` has show that Alice ignores the second input and Bob ignores the first input, Dr. Shell expressed concern over it being secure.
- The “active” bits effectively serve as the `isAlive` vector described in the paper, which helps the solver dynamically remove or keep clauses without revealing which ones.

---


```cpp
        // constructs a new Formula by conjoining phi_a and phi_b

        auto phi = phi_a->conjunction(phi_b);
        cout << "input formula: \n";

        // This actually prints out phi? The conjunction of the two
        // iterates through the clauses in phi, prints which clauses are active
        phi->print(true);

        // The solver constructor:
        // takes ph' and initializes the number of variables, 
        // decision states, model, heuristics, etc.
        // Also copies the formula to preserve the original input.
        Solver solver(nvar, phi);
```
```cpp
// further explaining the conjunction function
unique_ptr<Formula> Formula::conjunction(unique_ptr<Formula> const &f1) const{
	assert(f1->cls[0]->nvar == this->cls[0]->nvar);
	cout << f1->cls.size() << this ->cls.size() << endl; 
	vector<unique_ptr<Clause>> res_cls; 
    //push all clauses from f1 into res_cls
	for (int i = 0; i < f1->cls.size(); i ++)res_cls.push_back(f1->cls[i]->copy());
    //We also push all clauses from this formula into res_cls.
	for (int i = 0 ; i < this->cls.size(); i ++) res_cls.push_back(this->cls[i]->copy());
	return make_unique<Formula>(res_cls, false); 
}
```

**Explanation**:  
- `phi_a->conjunction(phi_b)` merges the sets of clauses from both parties
- `phi->print(true)` outputs the entire set of clauses (with some debug info)
- The `Solver` object (`Solver solver(nvar, phi)`) prepares data structures to track assignments, heuristics for picking variables, and so on, all using secure bits underneath.
- Why do I believe this protects the privacy? The call to copy() for each clause only copies the underlying secure bits (EMP Bit objects)
- we move the secure representation of each clause without exposing anything about its content. Each clause’s internal structure (BIClause, which stores whether a literal is present/absent) remains hidden behind the EMP Bits.

---

## Solving and Model Output

```cpp
        // The main loop of the slver
        // it iterates at most "number_of_steps" times
        // returns a model of variable assignments
        auto model = solver.solve(number_of_steps, false);
        cout << "model\n"; 
        cout << model->toString() << endl;

        delete io;
    }
```

**Explanation**:  
- `solver.solve(...)` is where the SAT algorithm as explained by 4.2



### Additional Security Commentary

- The code sets up a secure channel via NetIO, and setup_semi_honest() from EMP ensures that both parties communicate securely. Alice garbles the circuit, Bob evaluates, and they exchange just enough information to produce the final result.  
-  The active array inside Formula aligns with the isAlive vector from the paper. Setting it to 1 means the clause is still considered in the solver. Setting it to 0 means it is effectively removed.
- The conjunction function merges Alice’s and Bob’s formulas without revealing any sensitive detail of either side’s formula. The two sets of clauses remain in secure EMP Bit vector form until the solver needs to operate on them.  
- **Solver structure**: The solver uses a mix of secure data structures (`BIClause`, etc.) to handle each variable’s assignment, each clause’s status, and all the typical steps of a SAT solver. Every logical operation is done via the underlying garbled circuit computations, preventing data leaks in the semi-honest model.

