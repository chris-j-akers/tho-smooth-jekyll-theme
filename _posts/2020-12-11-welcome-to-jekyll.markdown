---
layout: post
title: "experimenting with merkle trees: the root hash of english"
description: "Building a merkle tree out of all the words in English get a root hash of the English Language"
author: Chris Akers
draft: true
published: true
---
## introduction
A long time ago, before I became more inimate with Merkle Trees, I had only heard about them in passing and assumed they were some sort of horrific mathematical construct. Of course, they're not, they're really just a type of binary tree.

As with most stuff like this, it's best to get acquainted by actually trying to implement one. This post journals a small program in `c` that reads in a fairly meaty set of 'data blocks' (over 450,000) and uses them to build a Merkle Tree and get a 32-byte root hash. The choice of 'data blocks', basically all the words in the English language, means tthat he program is easy to watch and debug so you can see exactly what is happening when it executes.

## merkle trees

There are plenty of articles out there that describe Merkle Trees, so I'm not going to detail here. A couple of good ones are the [codementor article](https://www.codementor.io/blog/merkle-trees-5h9arzd3n8) and I also really liked the one at [nakamoto.com](https://nakamoto.com/merkle-trees/) because it uses an example of transmitting a large file in blocks. Blockchain uses Merkle Trees heavily so you will find a lot of articles about Merkle Trees are based around blockchain and it was nice to see a different type of example.

To construct a Merkle Tree a dataset must be split into blocks and a hash taken from each of those blocks. These hashes are then stored in a list of tree nodes that will go on to form the bottom layer, or the leaves, of the tree. This layer is then split into pairs of nodes and the hash digests stored in each left and right pair node concatenated. These concatenated values are then hashed again, and this hash forms the data point for a node in the layer above, and in between, the pair. Each layer of the tree is built in this fashion until there’s only one node left, the root node. The data in this root node contains the digest of the entire tree.

### why?

In a decentralised peer-to-peer system, several distinct parties must interact with each other, but do not necessarily trust each other. They keep a copy of a large dataset and, for their interactions to be successful, each copy must be the same, down to the last byte. These datasets are updated by the parties at irregular intervals throughout the day.

How can everyone verify they have the same dataset?

A simplistic way is for each party to take a copy of everyone else's dataset and compare it with their own byte-by-byte. But what if there's thousands of parties? What if the data is sensitive and you don't want it transferred over some network? What if the data changes rapidly and you must go through this process hundreds, or even thousands, of times a day? How big is the data? What's the bandwidth cost on this?

## our input data

## running the program

### changing the data

- adding a word
- removing a word
- modifying a single letter in one of the words

Even changing a single letter in the input data of over 455,000 words completely changes the hash.

## c source

### main()
We start with `main()` and some option processing with `getops()`. The program uses the `cakelog` logger ([source](https://github.com/chris-j-akers/cakelog)) which outputs timestamped information to a log file, but we keep this optional as, obviously with the size of some our test data, logging slows the program down, especially if we force it to flush as with the `f` option.

We build our pointer-chain of leaves by calling the `build_leaves()` function on line #66 and then initiate the build of our merkle tree with `build_merkle_tree()`.

Finally, we print the returned root hash out to the console.

```c
int main(int argc, char *argv[]) {

    int opt;
    
    while ((opt = getopt(argc, argv, "df")) != -1) {
        if ((unsigned char)opt == 'd') {
            /* debug without flush */
            cakelog_initialise(argv[0], false);
        }
        else if ((unsigned char)opt == 'f') {
            /* debug with flush */
            cakelog_initialise(argv[0], true);
        }
        else {
            printf("Usage: %s [-d] <datafile>\n", argv[0]);
            exit(EXIT_FAILURE);
        }
    }

    if (optind >= argc) {
        printf("missing filename (Usage: %s [-d] <datafile>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    char *words = read_data_file(argv[optind]);
    long word_count = get_word_count(words);

    printf("building leaves\n");

    Node **leaves = build_leaves(words);
    Node *root = build_merkle_tree(leaves, word_count);
    printf("Root digest is: %s\n", root->sha256_digest);

    cakelog_stop();

}
```
### struct Node
To build out tree we need nodes and that means a `Node` struct. There should be no surprises here. A `Node` struct contains recursive `left` and `right` references to itself for the branches (or `NULL` if a leaf) and the data itself which will be the hash.

```c
struct Node {
    struct Node * left;
    struct Node * right;
    char *sha256_digest;
}; 

typedef struct Node Node;
```
We also have a handy constructor that builds and returns a pointer to a new `Node` struct.
```c
Node* new_node(Node *left, Node *right, char *sha256_digest) {

    cakelog("===== new_node() =====");
    cakelog("left: %p, right: %p, hash: [%s]", left, right, sha256_digest);

    Node * node = malloc(sizeof(Node));
    node->left = left;
    node->right = right;
    node->sha256_digest = sha256_digest;

    cakelog("returning new node at address %p", node);
    
    return node;
}c
```
### read_data_file()
We need to load our file of words into memory so we can build the tree out of them. `read_data_file()` does exactly this and returns a pointer to the memory buffer where they're eventually stored.

System calls are used to obtain a file-descriptor for the data-file ([`open()`](https://man7.org/linux/man-pages/man2/open.2.html){:target="_blank"}: line #5), get the size of the file in bytes so we can establish how much memory we need to allocate for our buffer ([`fstat()`](https://man7.org/linux/man-pages/man2/fstat.2.html){:target="_blank"}: line #16) and, finally (after allocating the buffer on line #27), load in the file contents. ([`read()`](https://man7.org/linux/man-pages/man2/read.2.html){:target="_blank"}: line #30).

One thing to note is that, before we use the file size as the `size` parameter our `malloc()` call, we must increment it by 1 (line #26). This is to provide a slot at the end of the buffer for the `\0` (`NULL` terminator) which is added on line #40.

```c
char* read_data_file(const char *dict_file) {

    cakelog("===== read_dictionary_file() =====");

    const int dictionary_fd = open(dict_file, O_RDONLY);
    if (dictionary_fd == -1) {
        perror("open()");
        cakelog("failed to open file: '%s'", dict_file);
        exit(EXIT_FAILURE);
    }

    cakelog("opened file %s", dict_file);

    struct stat dict_stats;

    if (fstat(dictionary_fd, &dict_stats) == -1) {
        cakelog("failed to get statistics for dictionary file");
        perror("fstat()");
        exit(EXIT_FAILURE);
    }

    const long file_size = dict_stats.st_size;

    cakelog("file_size is %ld bytes", dict_file, file_size);

    const long buffer_size = file_size + 1;
    char* buffer = malloc(buffer_size);

    ssize_t bytes_read;
    if((bytes_read = read(dictionary_fd, buffer, file_size)) != file_size) {
        cakelog("unable to load file");
        perror("read()");
        exit(EXIT_FAILURE);
    }

    cakelog("loaded %ld bytes into buffer", bytes_read);

    close(dictionary_fd);

    buffer[file_size] = '\0';

    return buffer;

}
```
### sha256()
`sha256()` generates a hash from the `char*` provided in the `buffer` parameter and returns a new `char*` to that hash.

To generate hash values, we use the OpenSSL [EVP](https://wiki.openssl.org/index.php/EVP) (or Digital EnVeloPe) interface which offers a high-level way to interact with the OpenSSL cryptography library. Having said that, it's not brilliantly documented beyond the man pages, so using it means scouting the internets for various examples and explanations. The man pages are a great reference, but you need more when it comes to stiching everything together. I also noted that there are many examples out there using OpenSSL internal calls directly, but the EVP interface is recommended by the OpenSSL organisation.

To compile a program using the OpenSSL functions you will obviously need the libraries installed first, but you will also need to add the compiler switches `-lssl` and `-lcrypto` to link them.

We start on line #8 where we declare a new `EVP_MD_CTX` (Digital EnVeloPe Message Digest Context) pointer (`mdctx`). Then on line #12 we use the `EVP_MD_CTX_new()` function to allocate space for the `EVP_MD_CTX` object.

On line #13 we set-up our `EVP_MD_CTX` to use the `EVP_sha256()` hash function by calling `EVP_DigestInit_ex()`.

On line #16 `EVP_DigestUpdate()` is passed a pointer to our `EVP_MD_CTX` along with the data we want to hash and its length. This function could be used multiple times to repeatedly update the hash in our `EVP_MD_CTX` until we're ready to receive it (for instance, if we're using a stream of socket data to generate the hash) but, here, we only call it once because we have all the data we need in the `data` parameter.

On line #19 we allocate some memory in our `hash_digest` variable ready to receive our hash result. We use the OpenSSL version of `malloc()` which is called, funnily enough, `OPENSSL_malloc()` which allocates memory for `EVP_MD` data-structures. The function `sha_256()` simply returns a SHA256 version of the `EVP_MD` datastructure and the `EVP_MD_size()` function wrapped around it returns the size of that data structure (a kind of `sizeof` function).

Finally, `EVP_DigestFinal_ex()` takes the hash-result from our MDCTX and copies it to our `hash_digest` variable that was pre-allocated on line #19 (described above). On line #24 we clear everything up (`EVP_MD_CTX_free`) and then return `hash_digest` back to the caller.
```c
unsigned char* sha256(const char *data) {

    cakelog("===== sha256() =====");

    unsigned int data_len = strlen(data);
    unsigned char *hash_digest;

    EVP_MD_CTX *mdctx;

    cakelog("initialising new mdctx");

    mdctx = EVP_MD_CTX_new();
    EVP_DigestInit_ex(mdctx, EVP_sha256(), NULL);
    
    cakelog("updating mdctx digest with data [%s]", data);
    EVP_DigestUpdate(mdctx, data, data_len);
    
    cakelog("initialising new hash_digest buffer");
    hash_digest = OPENSSL_malloc(EVP_MD_size(EVP_sha256()));
    
    EVP_DigestFinal_ex(mdctx, hash_digest, &data_len);
    cakelog("succesfully copied new digest to hash_digest buffer");
    
    EVP_MD_CTX_free(mdctx);
    cakelog("successfully freed mdctx digest");

    return hash_digest;

}
```
### hexidigest()
The hashing functions we use return the hash in the form of an `unsigned char*` - a set of 32 bytes that make up our hash-digest. In order to print our digest out into that familiar string of hexadecimal symbols we need to convert it to a `char*`, a C string, of ascii characters that can be passed to `printf()` or similar.

One thing to consider is that a hexadecimal byte is typically written as two hexidecimal digits (e.g. 0x2 is usually printed as 02), so this means we need to double the size of our storage to fit each byte into two ascii characters - see the `malloc()` call on line #5.

We build the string in a loop by converting each byte into its two-digit hexadecimal representation (line #7-#9) and adding it to our `hexdigest` buffer one at a time. We use a bit of pointer arithmetic to locate the next empty space in our buffer set the next two digits using `sprintf` with hex forat specifier. 

The constant `SHA256_DIGEST_LENGTH` is defined in the OpenSSL `sha.h` header. At the time of writing, it is 32.

```c
char* hexdigest(const unsigned char *hash) {

    cakelog("===== hexdigest() =====");

    char *hexdigest = malloc((SHA256_DIGEST_LENGTH*2)+1);

    for (int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
        sprintf(hexdigest + (i * 2), "%02x", hash[i]);
    }

    hexdigest[64]='\0';

    cakelog("returning %s", hexdigest);

    return hexdigest;

}
```
### get_word_count()
A simple function to count all the words in our data buffer. It does this by scanning along the entire buffer and incrementing the counter each time it finds an `\n` (newline) character.

It increment the counter one final time when the `for` loop finishes because the last word in the buffer is followed by `\0` (NULL terminator), not `\n`.
```c
long get_word_count(const char *data) {

    cakelog("===== get_word_count() =====");

    long word_count = 0;
    for (int i=0; data[i] != '\0'; i++) {
         if (data[i] == '\n') {
             word_count++;
         }
     }

     word_count++;

     cakelog("returning word count of %ld", word_count);
     
     return word_count;
}
```

### build_leaves()
`build_leaves()` creates `Node` structs around the SHA256 hash digest of each word it finds in our data buffer, giving us the bottom layer (or leaves) of our tree. It returns a chain of pointers (pointer-to-pointer structure) to each `Node`.

Because the list of words in our data is of a fixed length and, thanks to the way they're separated, easy to count, I can pre-allocate the memory required to store all my `Node` pointers. On line #5, then, we call `get_word_count()` to establish how may words there are in the data and use this value on line #6 to allocate enough memory for all the pointers in the chain.

Words are pulled out of the buffer using [`strtok()`](https://man7.org/linux/man-pages/man3/strtok.3.html){:target="_blank"} (initial call line #16, subsequent calls line #26) which splits a block of `char` data into it's component words (or tokens), according to the delimeters we provide (in this case `\n` for each word and `\0` to get the last word in the buffer).

Line #22 is where we build each `Node`. We're building the bottom layer of the tree, so both the `left` and `right` branches are set to `NULL`. To get the hash of each word we call the `sha256()` function, described above, which wraps calls to the `openssl` library own SHA256 hashing functions. We further wrap this in a call to `hexdigest()` also described above, so we can get the text representation of the hash.
```c
Node** build_leaves(char* buffer) {

    cakelog("===== build_leaves() =====");

    long word_count = get_word_count(buffer);
    Node **leaves = malloc(sizeof(Node*)*word_count);

    cakelog("allocated %ld bytes for %ld leaves (number of leaves * sizeof(pointer) + NULL terminator", word_count * sizeof(unsigned char*) + 1, word_count);

    long index = 0;
    long hash_count = 0;
    Node *n;

    cakelog("beginning loop through buffer using strtok");

    char *word = strtok(buffer, "\n\0");

    while( word != NULL) {

        cakelog("next word is [%s]", word);

        n = new_node(NULL, NULL, hexdigest(sha256(word)));
        leaves[index] = n;
        hash_count++;
        index++;
        word = strtok(NULL, "\n\0");

    }

    cakelog("returning %ld leaves", index);

    return leaves;
}
```
### build_merkle_tree()
`build_merkle_tree()` builds our Merkle Tree recursively, layer by layer from the bottom up. It returns a pointer to the `Node` at the root of the tree.

The `previous_layer` parameter is a pointer-chain of `Node` objects that will be used to build the next layer of nodes. The first time this function is called, `previous_layer` will contain the leaves, or bottom layer, of our tree.

The `previous_layer_len` parameter contains the number of `Node` pointers in `previous_layer`. This value is used to calculate the amount of memory needed when allocating our new layer.

The first thing we do on line #5 is check to see if `previous_layer_len` is `1`. If it is, then we already have the root of our tree and can simply return the `Node*` at the head of the chain.

Next we allocate enough memory to store all the pointers in our new layer of nodes. To do that we need to know how many nodes there will be. We're building a [Pefect Binary Tree](https://www.programiz.com/dsa/perfect-binary-tree), so expect new layers to have half the number of nodes of their previous layer. In theory, then, we should be able to divide `previous_layer_len` by `2` and use the result in our call to `malloc()`. But, hang on......What if `previous_layer_len` is an odd number? Say, `5`? In C, integer division rounds down, so `5 / 2 = 1`. We need the answer to be `2`, or rounded up, to ensure we create enough slots in our new layer (remember, if there is an odd number of leaves, the last, orphaned leaf is duplicated so it forms both the left and right branches of the node above it).

On line #12, then, we divide `previous_layer_len` by two but also round up the result using the `ceil()` function from the `Math.h` library. We've now got a figure we can use to allocate enough memory on line #13.

The bulk of the function is a `while` loop which starts on line #25. Two integers are used as indexes to pull out adjacent nodes in `previous_layer`. These nodes go on to form the `left` and `right` branches of a new `Node*`. The hash-digests of both nodes are concatenated and the result is passed to our `sha256()` function to get a brand new hash-digest. This hash-digest is then stored into the `sha256_digest` field of our new `Node*`.

Note that we convert hashes as they are returned from `sha256()` to `char*` using `hexdigest()` (from `unsiged char*`). For real-world usage this is not necessary, but it's useful when you're experimenting so log and trace files can be cross-checked with expected results.

The `if` statement is a check to make sure that both left and right `Node*` are available. If `previous_layer` has an odd number of `Node*` then the final iteration of the `while` loop will only be able to pull out a valid left `Node*`, there won't be a right node. As, above, this means we duplicate that `Node*` for both the `left` and `right` pointers in the new `Node*` and concatentate its hash-digest with itself before passing it to `sha256()`.

Line #207 is where we make the recursive call back into the function, but this time passing the newly created `next_layer` which will become the `current_layer` of the function's next iteration ... and so on.
```c
Node* build_merkle_tree(Node **previous_layer, long previous_layer_len) {

    cakelog("===== build_merkle_tree() =====");

    if (previous_layer_len == 1) {

        cakelog("previous_layer_len is 1 so we have root. Returning previous_layer[0] at address %p", previous_layer[0]);

        return previous_layer[0];
    }
   
    long next_layer_len = ceil(previous_layer_len/2.0);
    Node **next_layer = malloc(sizeof(Node*)*next_layer_len);

    cakelog("allocated space for %ld node pointers in next_layer at address %p", next_layer_len, next_layer);
    printf("allocated space for %ld node pointers in next_layer at address %p\n", next_layer_len, next_layer);
    
    int next_layer_index = 0;
    long previous_layer_left_index = 0;
    long previous_layer_right_index = 0;
    
    Node *n;
    char *digest = malloc(sizeof(char) * 129);

    while (previous_layer_left_index < previous_layer_len) {

        cakelog("top of loop");

        previous_layer_right_index = previous_layer_left_index + 1;

        if (previous_layer_right_index < previous_layer_len) {

            cakelog("both left node and right node available");
                       
            cakelog("left node addr: %p, left node hash: [%s], right node addr: %p, right node hash: [%s]", previous_layer[previous_layer_left_index], previous_layer[previous_layer_left_index]->sha256_digest,  previous_layer[previous_layer_right_index], previous_layer[previous_layer_right_index]->sha256_digest);
            
            strcpy(digest, previous_layer[previous_layer_left_index]->sha256_digest);
            strcat(digest, previous_layer[previous_layer_right_index]->sha256_digest);

            cakelog("concatenated digest is: %s", digest);

            n = new_node(previous_layer[previous_layer_left_index], 
                         previous_layer[previous_layer_right_index], 
                         hexdigest(sha256(digest)));

        }

        else {
        
            cakelog("only have left node available");
            
            cakelog("left node addr: %p, left node digest: [%s]", previous_layer[previous_layer_left_index], previous_layer[previous_layer_left_index]->sha256_digest);
                     
            strcpy(digest, previous_layer[previous_layer_left_index]->sha256_digest);
            strcat(digest, previous_layer[previous_layer_left_index]->sha256_digest);

            cakelog("new node concatenated digest is: %s", digest);

            n = new_node(previous_layer[previous_layer_left_index], 
                         previous_layer[previous_layer_left_index], 
                         hexdigest(sha256(digest)));
        }

        next_layer[next_layer_index] = n;
        
        cakelog("added node at address %p to next_layer with an index of %ld", n, next_layer_index);
        
        next_layer_index++;
        previous_layer_left_index = previous_layer_right_index + 1;
    }
    
    return build_merkle_tree(next_layer, next_layer_index);
}
```