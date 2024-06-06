# choose your compiler, e.g. gcc/clang
# example override to clang: make run CC=clang
CC = gcc

# the most basic way of building that is most likely to work on most systems
.PHONY: run
run: run.c
	$(CC) -O3 -o run run.c -lm

# useful for a debug build, can then e.g. analyze with valgrind, example:
# $ valgrind --leak-check=full ./run out/model.bin 1.0 3
rundebug: run.c
	$(CC) -g -o run run.c -lm
	
# includes model & tokenizer
.PHONY: emscripten
emscripten: ort.c
	emcc -c ort.c -o ort.o