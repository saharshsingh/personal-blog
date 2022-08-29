+++
author = "Saharsh Singh"
title = "Trie to Find Words"
slug= "trie-to-find-words"
date = "2019-06-18"
description = "In this article, I talk about the trie data structure and use it to create a solver for a word puzzle game I play on my mobile phone. In the process of trie-ing to find words (get it!), we also see a simple example of breadth first search at play."
tags = [
    "algorithms",
    "code",
    "data structures",
    "howto",
    "javascript",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2019/06/18/trie-to-find-words/",
]
+++

In this article, I talk about the trie data structure and use it to create a solver for a word puzzle game I play on my mobile phone. In the process of trie-ing to find words (get it!), we also see a simple example of breadth first search at play.

<!--more-->

A few years ago, I was a going through a technical interview. One of the questions involved something around processing a list of names. I forget the specifics, and I should not be divulging the details anyways. I came up with a solution that the interviewer liked, but I could tell it would not have been his preferred approach. Since we had a little time left over before the next interviewer, I asked him what would be a better way to solve this. He didn’t go into details, but said that this problem is a good use case for [tries](https://en.wikipedia.org/wiki/Trie). I hadn’t heard of tries before, but after the interview I went home and looked into them briefly. A trie is an ordered tree data structure that stores words so that they can be searched efficiently. Tries, the name being a play on the words **tree** and **retrieval**, are commonly used to implement the logic behind spell checkers and auto complete functionality. While it was fun learning about them, I never had a reason to implement one directly in any of my projects.

Fast forward to 2019. I have been hooked on a word puzzle game that I play on my cell phone to kill time here and there. The game essentially gives you a bunch of letters and a crossword grid. Your job is to fill up the crossword grid with words that can be formed from the given letters. Typically you are forming words that are three to six letters long. Well I was playing the game on a lazy Sunday afternoon, and while I playing I suddenly had a flashback to tries. I realized the trie data structure would be the perfect tool to build a solver for this game. I spent the next two hours white-boarding and coding a quick and dirty Java app. Since this was just an intellectual curiosity, I didn’t bother packaging up the app or exposing it outside my laptop.

A few days later, I was playing the game again in the backseat of a car and started missing my app whenever I would get stuck. I also realized this would be a good way to stop procrastinating and get going on my goal to start hosting apps and custom content on my personal AWS account that are exposed via the [saharsh.org](http://saharsh.org/) domain. Also, needing the app exposed over the web would be a good excuse to dust off my ReactJS skills. And boom! [words.saharsh.org](http://words.saharsh.org/) was born.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/1.png" >}}

The app is simple. You select the minimum and maximum length of words you want generated. Then you enter letters. The number of letters needs to equal the maximum length. As soon as you have entered as many letters as the maximum length, all the possible words are generated and spit out. After initial load the app does not need to make any server side calls and is pretty fast as a result. So how does this work? Let’s dig in. By the way, all the code for this app is [in this Github repo](https://github.com/saharsh-samples/trie-to-find-words).

On initial page load, the app takes a list of all English words and stores them in a trie data structure. Once all the letters are entered by the user, an algorithm is run on the trie that now contains all English words. The algorithm takes three inputs – the letters, minimum word length, and maximum word length – and spits out a list of generated words as an output.

Let’s add more detail. To keep things simple, let’s pretend all of English is made of the following five words:

* ace
* core
* corn
* cow
* cower

My trie implementation ([see code](https://github.com/saharsh-samples/trie-to-find-words/blob/master/src/impl/Trie.js)) is fairly brief. The structure is made of **nodes**. Each **node** has a **value** (a letter), **a boolean flag** indicating whether the node terminates a word, and **a mapping** of letters to children nodes. The root node of the structure is special. It does not have a value or a flag, just the map of children nodes. The fact that children of each node are stored in a map instead of an array or list is an optimization on my part. A typical implementation of a trie allocates a 26 element array for each node to store pointers to children. Each index in the array corresponds to a specific letter in the alphabet (e.g. `0->a`, `1->b`, .. , `25->z`), essentially creating an implicit key. I opted to use maps instead to increase code readability and reduce space complexity.

There are two functions bound to the trie structure – `insert` and `contains`. Only the `insert` function is really needed since my “words generator” algorithm starts at the root node and does a **breadth first search** of the trie structure. I put in a `contains` method to make unit testing easier. Following diagram shows a visual representation of my trie structure after inserting all five words from our mini-English dictionary.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/2.png" >}}

Obviously, the actual app uses a much more comprehensive list of English words, and the resulting trie contains many more nodes and many more mappings per node. Great! Now that our trie is loaded with all English words, let’s see how the algorithm to generate words from it works ([see code](https://github.com/saharsh-samples/trie-to-find-words/blob/master/src/impl/Impl.js)). As you can see in the screenshot of the app above, you feed three inputs into the algorithm:

* `minLength`: minimum number of characters generated words must have
* `maxLength`: maximum number of characters generated words must have
* `letters`: a list of letters, whose length equals maxLength, that words will be generated from

The algorithm, captured inside the `findWords` function, returns a list of words generated that meet the criteria provided via the inputs. So how is this list generated? Essentially, we do a [breadth first search (BFS)](https://en.wikipedia.org/wiki/Breadth-first_search) of the trie, visiting nodes containing letters in the input `letters` list. As we find nodes that complete words, we capture them in a list. Once all candidate nodes are examined (keeping in mind maximum length allowed for any word), the list capturing all the words we found is returned as the result. Let’s see this in action with some visuals.

For our example let’s pretend, given our 5-word English language from above, we want to find words that contain subsets of these letters – `c`, `o`, `w`, `e`, and `r`. So, we give the following inputs to our algorithm:

* `letters`: `[‘c’, ‘o’, ‘w’, ‘e’, ‘r’]`
* `minLength`: `3`
* `maxLength`: `5`

BFS is typically implemented using a queue – as opposed to [depth first search (DFS)](https://en.wikipedia.org/wiki/Depth-first_search) which is implemented using a stack. So the algorithm starts with creating this work queue. Since I am writing Javascript, where arrays are magical entities that can be anything you want them to be, I just need to allocate an array and use its `push` and `shift` functions. The queue is initialized with all of the nodes in the root mapping that contain a letter from the input `letters` list. In our case, there are two nodes in root mapping to be considered – `a` and `c`. Only `c` qualifies since `a` is not in the `letters` list. The algorithm also initializes an empty array to hold the words we will eventually find. So, at this point, following diagram represents what the state of algorithm’s execution looks like.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/3.png" >}}

You’ll notice the node in work queue is slightly different than the node in our trie structure. The node added to the queue is actually another abstraction that wraps around the original trie node. Let’s call this abstraction the candidate node. Each candidate node contains the associated trie node and a `crumbs` field. The crumbs field is an array that stores the path taken to the trie node by capturing indices from the input `letters` list. NOTE: In my visuals I show the actual letter to make the diagrams more intuitive. This information is used to quickly build the word when a node with `isWord` flag set to true is found. Also, it helps us with scenarios involving duplication of letters. For example, given `[a, c, o, r, r, t]` as the input list and a trie instance containing `c->a->r->r->o->t`, the second `r` node would be added to the work queue. On the other hand, given [`a, c, o, r, t]` as the input list and the same trie instance, the second `r` node would not be added to the work queue.

With the initial queue populated, we enter the main stage of the algorithm. This whole stage is a loop that keeps executing while the work queue is not empty. In this stage, we deque and process the node at the head of the work queue. The processing involves two steps.

1. If the node completes a word, and that word matches the minimum length requirement, add it to the words found list.
1. If the size of the crumbs list is less than the maxLength input, add to work queue any child nodes that contain letters from the input letters list that aren’t already part of the crumbs list.

So, once we process the `c` node in our work queue, we find no completed word and one eligible child node. The state of the algorithm’s execution changes to the following.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/4.png" >}}

Similarly, when the `o` node is processed, two child nodes are found and they are both added to the work queue.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/5.png" >}}

Things get more interesting with the `w` node. This node completes the word `cow`. The word is added to the words found list. An eligible child node is also found and added to the work queue.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/6.png" >}}

The next node, `r`, does not complete a word. It does contain two child nodes, but only one is eligible given our input letters list. The `e` node is added to the work queue, while the `n` node is ignored.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/7.png" >}}

The next node, `e` with crumbs `[ c, o, w, e ]`, does not complete a word, but does contain the eligible node `r`.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/8.png" >}}

The next two nodes both complete words, and neither contains any child nodes. The algorithm works off these final two nodes. First the word `core` is found and added.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/9.png" >}}

Finally, the word is `cower` is found and added to words found list.

{{< figure class="img-responsive" src="/images/posts/trie-to-find-words/10.png" >}}

With the work queue finally empty, the algorithm exits the loop and finishes up by returning the words found list.

And that’s it! Till next time…
