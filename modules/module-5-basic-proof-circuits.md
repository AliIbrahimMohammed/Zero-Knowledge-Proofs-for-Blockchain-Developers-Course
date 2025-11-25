# Module 5: Writing Basic Proof Circuits in Noir

## Learning Objectives
By the end of this lesson, you will be able to:
- Write basic arithmetic constraint circuits in Noir
- Implement range checks to prove knowledge of bounded values
- Create circuits that prove knowledge of discrete logarithms
- Develop circuits for simple privacy-preserving computations
- Debug and optimize basic Noir circuits
- Test circuits with different input values

## Table of Contents
1. [Introduction](#introduction)
2. [Simple Arithmetic Circuit](#simple-arithmetic-circuit)
3. [Range Check Circuit](#range-check-circuit)
4. [Simple Cryptographic Hash Circuit](#simple-cryptographic-hash-circuit)
5. [Quadratic Residue Example](#quadratic-residue-example)
6. [Array Processing Circuit](#array-processing-circuit)
7. [Conditional Logic in Circuits](#conditional-logic-in-circuits)
8. [Practical Example: Private Voting Circuit](#practical-example-private-voting-circuit)
9. [Circuit Optimization Tips](#circuit-optimization-tips)
10. [Testing Your Circuits](#testing-your-circuits)
11. [Common Errors and Debugging](#common-errors-and-debugging)
12. [Knowledge Check](#knowledge-check)
13. [Summary](#summary)
14. [Exercises](#exercises)

## Introduction

Welcome to the hands-on module where you'll write your first zero-knowledge proof circuits using Noir! This is where theory meets practice, and you'll see how to implement the mathematical concepts we've discussed. We'll start with simple circuits and gradually build more complex ones, giving you a solid foundation in ZKP development.

Now that you understand Noir's syntax and concepts, it's time to create actual proof circuits. Each circuit you write will express a computational statement that can be proven without revealing the underlying data. This is where the magic of zero-knowledge really begins!

## Simple Arithmetic Circuit

Let's start with a basic circuit that proves knowledge of two numbers whose product equals a public value.

### Circuit: Proving Knowledge of Factors

```rust
// src/main.nr
fn main(private_a: Field, private_b: Field, public_product: pub Field) {
    // Constraint: private_a * private_b = public_product
    assert(private_a * private_b == public_product);
}
```

### How It Works:
1. The prover knows two secret values (private_a and private_b)
2. The verifier knows their product (public_product)
3. The circuit proves that the prover knows factors of public_product
4. The verifier learns that the prover knows the factors, but not what the factors are

### Test with Prover.toml:
```toml
private_a = "17"
private_b = "19" 
public_product = "323"  # 17 * 19 = 323
```

## Range Check Circuit

Range checks are fundamental in ZKP systems - they prove that a value lies within specific bounds without revealing the exact value.

### Circuit: Proving a Value is Between 0 and 255

```rust
// src/main.nr
fn main(secret_value: Field, dummy: pub Field) {
    // We use dummy here because every circuit needs at least one public output
    
    // Prove secret_value is between 0 and 255 (8-bit range check)
    // Method: decompose into 8 bits and reassemble
    
    // Get the bit decomposition
    let bits = std::array::u8_to_bits(secret_value);
    
    // Verify that the bits reconstruct the original value
    let mut reconstructed: Field = 0;
    let mut multiplier: Field = 1;
    
    for i in 0..8 {
        // Each bit must be 0 or 1
        let bit = bits[i];
        assert(bit * (bit - 1) == 0);  // Only satisfied when bit is 0 or 1
        
        reconstructed = reconstructed + bit * multiplier;
        multiplier = multiplier * 2;
    }
    
    // The reconstructed value must equal the original
    assert(reconstructed == secret_value);
    
    // Set dummy to something to have a public output
    assert(dummy == 0);
}
```

### More Efficient Range Check with Built-in Function

```rust
fn main(secret_value: Field, dummy: pub Field) {
    // This asserts that secret_value fits in 8 bits (0 to 255)
    secret_value.assert_mineq(255);
    
    assert(dummy == 0);
}
```

## Simple Cryptographic Hash Circuit

Let's create a circuit that proves knowledge of a preimage to a hash without revealing the preimage.

### Circuit: Proving Knowledge of SHA-256 Preimage

```rust
// Note: Implementation may vary based on backend support
fn main(preimage: Field, expected_hash: pub [Field; 4]) {
    // This is a simplified example - real hash circuits are more complex
    // In practice, you'd use Noir's standard library functions when available
    
    // For demonstration, let's use a simple arithmetic hash
    let computed_hash_0 = preimage * 3 + 42;
    let computed_hash_1 = preimage * 5 + 73;
    let computed_hash_2 = preimage * 7 + 11;
    let computed_hash_3 = preimage * 11 + 13;
    
    // Verify each component of the hash matches
    assert(computed_hash_0 == expected_hash[0]);
    assert(computed_hash_1 == expected_hash[1]);
    assert(computed_hash_2 == expected_hash[2]);
    assert(computed_hash_3 == expected_hash[3]);
}
```

## Quadratic Residue Example

A more complex example showing a mathematical property proof.

### Circuit: Proving a Number is a Quadratic Residue

```rust
fn main(secret_square_root: Field, public_square: pub Field) {
    // Prove that public_square is a quadratic residue
    // by showing knowledge of its square root
    
    // Constraint: secret_square_root * secret_square_root = public_square
    assert(secret_square_root * secret_square_root == public_square);
}
```

## Array Processing Circuit

Let's build a circuit that processes multiple values.

### Circuit: Proving Knowledge of a Secret Array with Public Sum

```rust
fn main(
    secret_array: [Field; 5], 
    public_sum: pub Field,
    public_product: pub Field
) {
    // Calculate sum of array elements
    let calculated_sum = secret_array[0] + secret_array[1] + 
                         secret_array[2] + secret_array[3] + secret_array[4];
    
    // Calculate product of array elements  
    let calculated_product = secret_array[0] * secret_array[1] * 
                            secret_array[2] * secret_array[3] * secret_array[4];
    
    // Verify the public values match calculations
    assert(calculated_sum == public_sum);
    assert(calculated_product == public_product);
}
```

### Test with Prover.toml:
```toml
secret_array = ["2", "3", "4", "5", "6"]
public_sum = "20"      # 2+3+4+5+6 = 20
public_product = "720" # 2*3*4*5*6 = 720
```

## Conditional Logic in Circuits

Noir supports conditional logic, which is crucial for more complex proofs.

### Circuit: Conditional Payment Proof

```rust
fn main(
    account_balance: Field, 
    payment_amount: Field,
    sufficient_balance: bool,
    new_balance: pub Field
) {
    if sufficient_balance {
        // If sufficient balance, new balance is original - payment
        assert(new_balance == account_balance - payment_amount);
        // Verify the payment is possible
        assert(account_balance >= payment_amount);
    } else {
        // If insufficient balance, new balance stays the same
        assert(new_balance == account_balance);
        // Verify the payment is not possible
        assert(account_balance < payment_amount);
    }
    
    // Prove knowledge of account balance and payment amount
    assert(account_balance > 0);
    assert(payment_amount > 0);
}
```

## Practical Example: Private Voting Circuit

Let's combine several concepts into a practical example: proving a valid vote was cast without revealing the vote.

### Circuit: Private Vote Proof

```rust
fn main(
    vote_id: Field,           // Secret identifier for the vote
    choice: Field,            // Vote choice: 0 or 1 (yes/no)
    voter_secret: Field,      // Secret key of voter
    vote_commitment: pub Field,
    election_id: pub Field
) {
    // Verify vote choice is 0 or 1
    assert(choice * (choice - 1) == 0);
    
    // Verify election ID constraint (arbitrary example)
    assert(election_id > 1000);  // Example constraint
    
    // A simple commitment scheme
    // In practice, you'd use proper cryptographic commitments
    let calculated_commitment = vote_id * 31 + choice * 17 + voter_secret * 13;
    
    // Prove the commitment is valid
    assert(vote_commitment == calculated_commitment);
    
    // Prove voter knows the secret
    assert(voter_secret > 0);
}
```

## Circuit Optimization Tips

### 1. Minimize Constraints

```rust
// Less efficient
fn inefficient_example(a: Field, b: Field, result: pub Field) {
    let temp1 = a * 2;
    let temp2 = b * 3;
    let temp3 = temp1 + temp2;
    assert(result == temp3);
}

// More efficient  
fn efficient_example(a: Field, b: Field, result: pub Field) {
    assert(result == a * 2 + b * 3);
}
```

### 2. Use Early Returns for Complex Logic

```rust
fn process_data(input: Field, result: pub Field) -> pub Field {
    // Early validation
    if input > 1000000 {
        assert(false); // Invalid input
    }
    
    // Process the data
    let processed = input * input + 42;
    assert(result == processed);
    
    processed
}
```

## Testing Your Circuits

### Setting Up Tests

Create a `Prover.toml` file to specify inputs:

```toml
# Example for the factors circuit
private_a = "17"
private_b = "19"
public_product = "323"
```

### Running Tests

```bash
# Compile the circuit
nargo check

# Generate a proof with specified inputs
nargo prove myproof

# Verify the proof
nargo verify myproof
```

### Testing Multiple Scenarios

Create different TOML files for different test cases:
- `Prover_valid.toml` - Valid inputs that should pass
- `Prover_invalid.toml` - Invalid inputs that should fail
- `Prover_edge_cases.toml` - Boundary cases

## Common Errors and Debugging

### Error: Unsatisfied Constraints

This happens when the inputs don't satisfy the circuit constraints:

```rust
// Problem:
fn main(x: Field, y: pub Field) {
    assert(x * x == y);  // If x=2 but y=5, this fails
}

// Solution: Ensure inputs satisfy all constraints
// In Prover.toml: x = "2", y = "4" (since 2*2=4)
```

### Error: Out of Range Values

```rust
// Problem: Field overflow
fn main(large_a: Field, large_b: Field, result: pub Field) {
    // This might overflow the field size
    assert(large_a + large_b == result);
}
```

## Knowledge Check

Think about these questions before continuing:
1. How does a range check circuit prove that a value is within specific bounds?
2. What's the difference between public and private inputs in terms of privacy?
3. How would you modify the private voting circuit to support more than 2 choices?

## Summary

In this module, you've implemented several types of basic proof circuits in Noir:
- Simple arithmetic constraint circuits
- Range check circuits for bounded values
- Circuits with conditional logic
- Multi-value processing circuits
- A practical example with the voting circuit

You also learned about testing, optimization, and debugging techniques. These basic circuits form the building blocks for more complex ZKP applications. In the next module, we'll explore advanced techniques for building more sophisticated proof systems.

## Exercises

### Exercise 5.1: Basic Arithmetic Practice
1. Write a Noir circuit that proves knowledge of the cube root of a number
2. Create a circuit that proves two numbers are coprime (their GCD is 1)
3. Implement a circuit that proves a number is prime (for small primes)

### Exercise 5.2: Range and Bounds
1. Write a circuit that proves a value is within a specific range using bit decomposition
2. Create a circuit that proves a value is a power of 2
3. Implement a circuit that proves a value is a Fibonacci number

### Exercise 5.3: Privacy Applications
1. Design a circuit for private authentication (proving you know a password without revealing it)
2. Create a circuit for private voting with multiple candidates (not just yes/no)
3. Implement a circuit for private balance checking (proving you have more than X without revealing exact balance)