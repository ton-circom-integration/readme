# TON circom integration

## Links

To see examples of what this project does, please check out
* [Generated contract](https://github.com/ton-circom-integration/example/blob/main/contracts/example.fc)
* [Usage example](https://github.com/ton-circom-integration/example/blob/main/tests/Example.spec.ts)

The main work in this project was to patch existing libraries to work with TON, here is their list (all rights belong to original authors):
* [snarkjs](https://github.com/ton-circom-integration/snarkjs)
* [ffjavascript](https://github.com/ton-circom-integration/ffjavascript)
* [wasmcurves](https://github.com/ton-circom-integration/wasmcurves)

## How to use

### Create the circuit

These steps correspond to steps 9-10 of [snarkjs's README](https://github.com/iden3/snarkjs)

1. Create the circuit source file:
```circom
// main.circom
pragma circom 2.0.0;

template Poly() {
    signal input a;
    signal output b;
    b <== a*a - 4;
}

component main = Poly();
```
Using this simple circuit, the prover can convince the verifier that they know the solution to the polynomial, in this case `a = 2`, without revealing the value of `a`. The verifier needs to check that the public output `b` is equal to 0 after they check that the proof itself is valid.

**WARNING:** the polynomial can be decoded from the R1CS (the constraint system) and thus the solution can be learned from the circuit itself, so this isn't a particularly safe or useful circuit and it is only used here for demonstration purposes.

2. Compile the circuit using the latest version of circom (2.1.5 at the time of writing):
```bash
circom main.circom -p bls12381 --r1cs --wasm --sym
```
We'l get files `main.r1cs`, `main.sym`, and the directory `main_js` to help us generate witnesses (execution traces with public inputs and outputs) for our circuit.

Installation of [circom](https://docs.circom.io/) is outside the scope of this document.

### Calculate the witness

These steps correspond to step 14 of [snarkjs's README](https://github.com/iden3/snarkjs)

1. Create the input file:
```
// input.json
{"a":"2"}
```
Please note that the numbers need to be encoded as strings because JS loses precision with non-string numbers when parsing JSON.

2. Generate the witness:
```bash
node main_js/generate_witness.js main_js/main.wasm input.json witness.wtns
```
`main_js` is the directory from step 2 of circuit creation, `witness.wtns` is the output file.

### Get the patched version of snarkjs

[snarkjs](https://github.com/iden3/snarkjs) is a tool to interact with ZK circuits. However, the linked version lacks the capabilities needed to work with TON, and so the [patched version](https://github.com/ton-circom-integration/snarkjs) was created, and is the main part of the work done for this integration.

To get the patched version, you can do one of the following:
* Clone the patched version and optionally build it yourself, then use the tool as follows: `node path-to-patched-version/build/cli.cjs`
* Or, `npm install -g @krigga/snarkjs` and use it as `snarkjs` (this will likely conflict with the original version if you have it installed)

In this document, the tool will be used as `snarkjs`, but `node path-to-patched-version/build/cli.cjs` can be substituted for it if you cloned the tool instead of installing it.

### Perform the trusted setup ceremony

These steps correspond to steps 1-7 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs powersoftau new bls12-381 4 pot4_0000.ptau -v # depending on your circuit, you might need more powers than 4, but for our example circuit 4 is enough
snarkjs powersoftau contribute pot4_0000.ptau pot4_0001.ptau --name="First contribution" -v
# we'll only make 1 contribution here for simplicity, however note that the trusted setup ceremony is a very important process, and there must be at least one contribution by a non-malicious party for the setup to be safe
snarkjs powersoftau verify pot4_0001.ptau
snarkjs powersoftau beacon pot4_0001.ptau pot4_beacon.ptau 0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final beacon"
powersoftau prepare phase2 pot4_beacon.ptau pot4_final.ptau -v
```

### Setup the ZK keys

This step corresponds to steps 15 and 22 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs plonk setup main.r1cs pot4_final.ptau circuit_final.zkey
snarkjs zkey export verificationkey circuit_final.zkey verification_key.json
```

### Generate the proof

This step corresponds to step 23 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json -v -transcript="sha256"
```

### Verify the proof

This step corresponds to step 24 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs plonk verify verification_key.json public.json proof.json -transcript="sha256" -v
```

### Export the FunC verifier

This step corresponds to step 25 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs zkey export funcverifier circuit_final.zkey verifier.func
```

Please note that any additional checks on public inputs and outputs (such as the check `b == 0` for our sample circuit) need to be implemented manually, the generated verifier code only checks the validity of the proof given the provided public inputs and outputs.

### Export verifier call arguments

This step corresponds to step 26 of [snarkjs's README](https://github.com/iden3/snarkjs)

```bash
snarkjs zkey export funccalldata public.json proof.json
```

This command generates an array of `TupleItem`s (a type from `ton-core`) that can be passed to the get method in the verifier smart contract. Check out the [usage example](https://github.com/ton-circom-integration/example/blob/main/tests/Example.spec.ts)

The format in which to pass proof data in Cells (for use in internal/external messages for example) shall be defined by the developer of the end contract and is not defined by the author of this integration.
