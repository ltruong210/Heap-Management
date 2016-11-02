#include <stdio.h>
#include <stdlib.h>
#include <string.h>


#define HSIZE	2
#define WSIZE	4  
#define PACK(size, alloc)	((size) | (alloc))
#define GET(p)	(*(unsigned short *)(p))
#define PUT(p, val)	(*(unsigned short *)(p) = (val))
#define GET_SIZE(p)	(GET(p) & ~0x1)
#define GET_ALLOC(p)	(GET(p) & 0x1) 
#define GET_BLKN(p)	(GET(p + HSIZE))
#define HDRP(bp)	((char *)(bp) - WSIZE)
#define NEXT_BLKP(bp)	((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))

void *mm_malloc(unsigned short size);
void *mm_free(unsigned short block_number);
void *find_fit(unsigned short asize);
void *place(void *bp, unsigned short asize);
void *write_to(unsigned short block_number, char char_to_write, unsigned short write_times);
void *print_from(unsigned short block_number, unsigned short print_length);
unsigned short corruption(char *bp);

unsigned short MAX_LENGTH = 400;
char *heap = NULL;
char *heap_start = NULL;
char *max_addr = NULL;
unsigned short current_block_number = 0;

int main() {
	heap = (char *) malloc(MAX_LENGTH);
	heap_start = heap + 2 * WSIZE;
	max_addr = (char *) (heap + MAX_LENGTH - WSIZE);
	PUT(heap, 0); 
	current_block_number = 1;
	unsigned short max_block_size = MAX_LENGTH - 3 * WSIZE;
	PUT(heap + WSIZE, PACK(max_block_size, 0));
	PUT(heap + WSIZE + HSIZE, 0);
	PUT(max_addr, PACK(0, 1));

	while (1) {
		char command[MAX_LENGTH];
		memset(command, '\0', MAX_LENGTH);
        unsigned short index = 0;
        int current_char = EOF;
        printf("\n\n> ");
        while (current_char) {
                current_char = getchar();
                if (current_char == EOF || current_char == '\n') {
                        current_char = 0;
                } else if (index >= MAX_LENGTH) {
                	printf("\n\nExceeded max input length.\n");
                	break;
                } else {
                	command[index++] = current_char;
                }
        }

		char command_name[MAX_LENGTH];
		memset(command_name, '\0', MAX_LENGTH);
		char command_arg_1[MAX_LENGTH];
		memset(command_arg_1, '\0', MAX_LENGTH);
		char command_arg_2[MAX_LENGTH];
		memset(command_arg_2, '\0', MAX_LENGTH);
		char command_arg_3[MAX_LENGTH];
		memset(command_arg_3, '\0', MAX_LENGTH);
		sscanf(command, "%s%s%s%s", command_name, command_arg_1, command_arg_2, command_arg_3);

		if (strncmp(command_name, "allocate", 8) == 0) {
			int size = atoi(command_arg_1);
			if (size <= 0) {
				printf("\nCan only allocate a positive number of bytes, %s is invalid", command_arg_1);
			}
			void *current_bp = mm_malloc((unsigned short) size);
			if (current_bp) {
				printf("\n%d", current_block_number++);
			} else {
				printf("\nOut of memory, can not allocate %d bytes.", size);
			}
		} else if (strncmp(command_name, "free", 4) == 0) {
			int block_number = atoi(command_arg_1);
			if (block_number <= 0) {
				printf("\nCan only free a positive block number, %s is invalid.", command_arg_1);
			} else if (block_number >= current_block_number) {
				printf("\nBlock number %d does not yet exist, can't free.", block_number);
			} else {
				mm_free((unsigned short) block_number);
			}
		} else if (strncmp(command_name, "blocklist", 9) == 0) {
			printf("\nSize\tAllocated\tStart\t\tEnd");
			char *bp;
			for (bp = heap_start; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
				char *allocated = "yes";
				if (!GET_ALLOC(HDRP(bp))) {
					allocated = "no";
				}
				unsigned short size = GET_SIZE(HDRP(bp));
				if (corruption(bp) == 1) {
					break;
				}
				printf("\n\n%d\t%s\t\t%p\t%p", size, allocated, HDRP(bp), HDRP(bp) + size - 1); 
			}
		} else if (strncmp(command_name, "writeheap", 9) == 0) {
			int block_number = atoi(command_arg_1);
			if (block_number <= 0) {
				printf("\nCan only write to a positive block number, %s is invalid.", command_arg_1);
			} else if (block_number >= current_block_number) {
				printf("\nBlock number %d does not yet exist, can't write to.", block_number);
			} else {
				char *char_to_write = command_arg_2;
				if (strlen(char_to_write) != 1) {
					printf("\nCan only write one character a time, %s is not one character.", char_to_write);
				} else {
					int write_times = atoi(command_arg_3);
					if (write_times <= 0) {
						printf("\nCan only write %s a positive number of times, %s is invalid.", char_to_write, command_arg_3);
					} else {
						write_to((unsigned short) block_number, char_to_write[0], (unsigned short) write_times);
					}
				}
			}
		} else if (strncmp(command_name, "printheap", 9) == 0) {
			int block_number = atoi(command_arg_1);
			if (block_number <= 0) {
				printf("\nCan only print from a positive block number, %s is invalid.", command_arg_1);
			} else if (block_number >= current_block_number) {
				printf("\nBlock number %d does not yet exist, can't print from.", block_number);
			} else {
				int print_length = atoi(command_arg_2);
				if (print_length <= 0) {
					printf("\nCan only print a positive print length, %s is invalid.", command_arg_2);
				} else {
					print_from((unsigned short) block_number, (unsigned short) print_length);
				}
				
			}
		} else if (strncmp(command_name, "quit", 4) == 0) {
			break;
		} else {
			printf("\nunknown command");
		}
		
	}

	free(heap);
	return 0;
}


void *mm_malloc(unsigned short size) {
    unsigned short asize;     
    void *bp;      

    if (size > 0) {
	    if (size <= WSIZE)                                         
			asize = 2 * WSIZE;                                     
	    else
			asize = WSIZE * ((size + WSIZE + (WSIZE - 1))/WSIZE);
	    if ((bp = find_fit(asize)) != NULL) {  
			place(bp, (unsigned short) asize);                  
			return bp;
	    }
	}

	return NULL;              
}


void *mm_free(unsigned short block_number) {
	char *bp;
	for (bp = heap_start; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
		if (corruption(bp) == 1) {
			return NULL;
		}
		if (HDRP(bp) && GET_ALLOC(HDRP(bp)) && GET_BLKN(HDRP(bp)) == block_number) {
			unsigned short size = GET_SIZE(HDRP(bp));
			PUT(HDRP(bp), PACK(size, 0));
			return (void *) bp;
		}
	}

	return NULL;
	
}


void *find_fit(unsigned short asize) {
	char *bp;
	for (bp = heap_start; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
		if (corruption(bp) == 1) {
			return NULL;
		}
		if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
			return (void *) bp;
		}
	}

	return NULL;
}


void *place(void *bp, unsigned short asize) {
	unsigned short leftover_size = GET_SIZE(HDRP(bp)) - asize;
	PUT(HDRP(bp), PACK(asize, 1));
	PUT(HDRP(bp) + HSIZE, current_block_number);
	if (leftover_size >= 2 * WSIZE) {
		bp = NEXT_BLKP(bp);
		PUT(HDRP(bp), PACK(leftover_size, 0));
		PUT(HDRP(bp) + HSIZE, 0);
	}

	return bp;
}


void *write_to(unsigned short block_number, char char_to_write, unsigned short write_times) {
	char *bp;
	for (bp = heap_start; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
		if (corruption(bp) == 1) {
			return NULL;
		}
		if (HDRP(bp) && GET_ALLOC(HDRP(bp)) && GET_BLKN(HDRP(bp)) == block_number) {
			unsigned short i;
			for (i = 0; i < write_times; i++) {
				if (bp == max_addr) {
					printf("\nReached end of heap, terminating further writing.");
					break;
				}
				*bp++ = char_to_write;
			}
			break;
		}
	}

	return (void *)bp;
}


void *print_from(unsigned short block_number, unsigned short print_length) {
	char *bp ;
	for (bp = heap_start; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
		if (corruption(bp) == 1) {
			return NULL;
		}
		if (HDRP(bp) && GET_BLKN(HDRP(bp)) == block_number) {
			printf("\n");
			unsigned short i;
			for (i = 0; i < print_length; i++) {
				if (bp == max_addr) {
					printf("\nReached end of heap, terminating further printing.");
					break;
				}
				printf("%c", *bp++);
			}
			break;
		}
	}

	return (void *)bp;
}


unsigned short corruption(char *bp) {
	unsigned short size = GET_SIZE(HDRP(bp));
	if (bp + size - max_addr > 0) {
		printf("\nCorruption detected, refusing operation to prevent crash.");
		return 1;
	} 

	return 0;
}
