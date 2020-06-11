# Symbolic Data Flow Analysis on Ghidra's P-Code IL

This code performs symbolic data flow analysis through P-Code's SSA (Static Single Assignment) form, and records loads and stores performed on the parameters to a function to identify its struct. Works on all architectures supported by Ghidra.

### Usage

After Ghidra's auto-analysis for a binary, navigate to the function you want to be analyzed, then run the python plugin file `go.py` as a Ghidra plugin through either Ghidra's script manager or Python console. Make sure the other python files are in Ghidra's script path as well.

### Key Ideas

- Symbolic IL (p-code) SSA representation analysis to identify stores and loads
- Interprocedural analysis through forwards and backwards analysis. Forwards analysis identifies the stores and loads performed on a function argument. Backward analysis relates the return value of a function to function arguments. These two ideas can utilize memoization.
- To find arrays, loop variant variables in the pcode are identified. Two runs are performed with loop variants having different initial conditions. If the members between two runs of a struct is different and stride is consistent, then that struct is an array.
- TODO: find subarrays within structs, instead of full arrays

### Current results (very much a work in progress)

In a toy program with the following structs

```c
typedef struct {
  char haha;
  int L;
  int L2;
  int L3;
} dec2;

typedef struct {
  char* buf;
  int length;
  dec2* lol;
  dec2* lol2;
} dec;

typedef struct {
  int return_code;
  int return_value;
  dec* buf;
} hack;
```

Results are stored in a binary expression tree, which can then be converted into struct representation.

Struct representation and harness recovered using `stable2`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <stdint.h>

struct S3 {
	char entry_0
	char entry_1[3] NOT ACCESSED
	uint32_t entry_2
	uint32_t entry_3
	uint32_t entry_4
}
struct S2 {
	char entry_0
	char entry_1[3] NOT ACCESSED
	uint32_t entry_2
	uint32_t entry_3
	uint32_t entry_4
}
struct S1 {
	uint8_t* entry_0
	uint32_t entry_1
	uint32_t entry_2 NOT ACCESSED
	S2* entry_3
	S3* entry_4
}
struct S0 {
	uint32_t entry_0
	uint32_t entry_1
	S1* entry_2
}

typedef int(*func)(void* a, ...);

int main(int argc, char** argv) {
	void* handler = dlopen("/D:/CTF/research/gearshift/case6", RTLD_LAZY);
	void* base = *((void**)handler);
	func f = (func)(base + 2264);

	FILE* h = fopen(argv[1], "r");

	S0* ARG0;
	ARG0 = (S0*)malloc(16);
	fread((char*)&ARG0->entry_0, 1, 4, h);
	fread((char*)&ARG0->entry_1, 1, 4, h);
	ARG0->entry_2 = (S1*)malloc(32);
	ARG0->entry_2->entry_0 = (char*)malloc(8);
	fread((char*)ARG0->entry_2->entry_0, 1, 8, h);
	fread((char*)&ARG0->entry_2->entry_1, 1, 4, h);
	fread((char*)&ARG0->entry_2->entry_2, 1, 4, h);
	ARG0->entry_2->entry_3 = (S2*)malloc(16);
	fread((char*)&ARG0->entry_2->entry_3->entry_0, 1, 1, h);
	fread((char*)&ARG0->entry_2->entry_3->entry_1, 1, 3, h);
	fread((char*)&ARG0->entry_2->entry_3->entry_2, 1, 4, h);
	fread((char*)&ARG0->entry_2->entry_3->entry_3, 1, 4, h);
	fread((char*)&ARG0->entry_2->entry_3->entry_4, 1, 4, h);
	ARG0->entry_2->entry_4 = (S3*)malloc(16);
	fread((char*)&ARG0->entry_2->entry_4->entry_0, 1, 1, h);
	fread((char*)&ARG0->entry_2->entry_4->entry_1, 1, 3, h);
	fread((char*)&ARG0->entry_2->entry_4->entry_2, 1, 4, h);
	fread((char*)&ARG0->entry_2->entry_4->entry_3, 1, 4, h);
	fread((char*)&ARG0->entry_2->entry_4->entry_4, 1, 4, h);
	

	int res = f((void*)ARG0);

		free(ARG0->entry_2->entry_0);
	free(ARG0->entry_2->entry_3);
	free(ARG0->entry_2->entry_4);
	free(ARG0->entry_2);
	free(ARG0);
	

	printf("Result: %d\n", res);
}
```