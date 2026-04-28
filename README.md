# Override.exe Round 1 – CTF Writeups

This repository contains my writeups for challenges from Override.exe Round 1 CTF.
All solutions are solved individually, focusing on understanding the logic rather than just extracting flags.

The goal of this repository is not just to document solutions, but to break down the thought process, methodology, and techniques used to approach each challenge.

## Repository Structure
```
├── forensics/
├── reverse/
├── misc/
└── README.md
```
Each folder contains writeups categorized by domain:

* forensics/ → File analysis, metadata inspection, steganography, memory dumps, etc.
* reverse/ → Binary analysis, decompilation, custom logic reversing
* misc/ → Logic-based challenges, scripting, or unconventional problems

## Approach

Each writeup follows a structured methodology:

* Challenge Overview
> Understanding what is given and what is expected.
* Initial Analysis
> Observations from files, binaries, or provided data.
* Tools & Techniques Used
 * strings
 * file
 * binwalk
 * Ghidra / IDA
 * CyberChef
 * Custom scripts (Python)
* Exploitation / Solving Process
> Step-by-step breakdown of how the challenge was solved.
* Flag Extraction
> Final steps leading to the flag.

## Key Highlights
* Focus on manual reasoning over blind tool usage
* Covers real-world techniques used in CTF competitions
* Includes challenges involving:
* Custom virtual machines
* Encryption/Decryption logic
* Hidden data extraction
* Binary reversing

## Why This Repository Exists

This repo serves as:

* A learning resource for beginners getting into CTFs
* A revision log for myself
* A proof of problem-solving ability in competitive environments

Some commonly used tools across challenges:

* Ghidra
* IDA Free
* CyberChef
* binwalk
* strings
* Python

## Notes
- All writeups are based on challenges from a public CTF.
- Flags are included for educational purposes.
- The focus is on learning and methodology, not just answers.

### Author

**Avaneesh Yajurvedi**
