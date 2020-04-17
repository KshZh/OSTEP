```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
	if (argc != 2) {
		fprintf(stderr, "Usage: ./a.out [amount of memory in megabytes]");
		exit(EXIT_FAILURE);
	}
	size_t num_bytes = atoi(argv[1])*1024*1024;
	char* p = malloc(num_bytes);
	for (size_t i=0; ; i=(i+1)%num_bytes) {
		p[i] = 1;
	}
}
```

man free

man pmap