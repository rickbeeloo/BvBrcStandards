# Host name standardization

### Tried things + solutions

If the host names are somewhat close to that in the NCBI taxonomy database **fuzzing** seems like a good option. Since the NCBI taxonomy database has over 4 million entries a simple search of our 6K vs 4M takes forever with traditional fuzzers, such as [fuzzyWuzzzy](https://github.com/seatgeek/fuzzywuzzy). There are alternatives such as [NeoFuzz](https://github.com/x-tabdeveloping/neofuzz) which are fast for general distance meassures such as a ratio check, however the [partial ratio check](https://medium.com/@chandu.bathula16/understanding-fuzzy-string-matching-exploring-fuzz-ratio-fuzz-partial-ratio-token-set-ratio-and-d6892430f53c#:~:text=Fuzz%20Partial%20Ratio%20is%20a,rather%20than%20the%20entire%20string.) is super slow. A partial fuzz is nice for contaminated text, like `Sick cow` should still match `Cow` with 100%. Since fuzzing, especially partials, is too slow I implemented n-gram matching in Rust, called [HeurFuzz](https://github.com/rickbeeloo/HeurFuzz). This is super fast for big searches but showed several other problems:
- Works well for typos and similar texts, such as "red-breasted nuthatch" ~ "white-breasted nuthatch", and "rhiino" ~ "rhino" 
- Fails for similar words with totally different meanings, such as `Python regius` and `Regius takahashii`, both share "Regius" but the former is a snake and the latter a fish
- A small set of typos can be a totally different species. For example `male` is easily matched to `malea`, but malea are sea shells (see [here](https://www.marinespecies.org/aphia.php?p=taxdetails&id=217025))

Fuzzing on one hand helps matching when typos are present, but on the other hand introduces a lot of wrong matches because of it. So we need something that only matches when typos *make sense*, using [sentence transformers](https://github.com/UKPLab/sentence-transformers) seems like a good choice. This can embed sentences with contextual information from corpuses the transformers are trained on. So coming back to the previous example, `male` would be embedded with words like `human`, `female`, etc, whereas `malea` would be embedded with words like `shell`, `sea`, `pacific`, etc. For this to work we need a transormers trained on biomedical data. General embeddings, like [Roberta](https://huggingface.co/sentence-transformers/all-roberta-large-v1) would work well on data from WikiPedia, this includes lots of taoxnomic entities. But it will fail for more specfialized biomedical information, see for example [BioBert paper](https://arxiv.org/pdf/1901.08746.pdf). Luckily [huggingface](https://huggingface.co/) has a large repo of sentence transformers, including a [few trained on PubMed](https://huggingface.co/models?pipeline_tag=sentence-similarity&sort=trending&search=pubmed). Specifically  `NeuML/pubmedbert-base-embeddings` and `tavakolih/all-MiniLM-L6-v2-pubmed-full`. What we learned from this:
- This works great for words that are similar/the same, such as the example of `snake` vs `snakes`
- hmm, but it still fails for `male` vs `malea`, it scores `male - malea` higher (`0.7`) than `male - human` (`0.3`) for example. This somewhat makes sense as male can be any species, but ideally we want this just to say "I dont know" then (i.e. very low scores). Yet on a scale of `0..1`, `0.7` seems really high for something that just "looks like it" and male is in no way directly related to a sea shell. 

So the question is what goes wrong there? This is mainly because sentence transformers are trained on sentences but now we give them single words. The problem then arises from the fact that the sentence transformer cannot leverage context to disambiguate words. Take `bank`, it could be `river bank` and `bank to bring money to`. If we gave those descriptoins instead the sentence transformer would work well. Just passing `male` or `malea` provides too little information for an accurate embedding, and hence creates erroneous mappings. Instead of sentence transformers we might have to go back to the simpler embeddings, such as [word2vec](https://en.wikipedia.org/wiki/Word2vec#:~:text=Word2vec%20is%20a%20technique%20in,text%20in%20a%20large%20corpus). Huggingface has a bunch of word2vec models, including those trained on WikiPedia such as [google news + wiki](https://huggingface.co/fse/word2vec-google-news-300). We already know these will lack field specific knowledge. So I downloaded all `26M` [pubmed abstracts from huggingface](https://huggingface.co/datasets/pubmed), and trained a Word2Vec model using [FinalFusion](https://github.com/finalfusion/finalfrontier) - a fast Rust based word embedder ( took ~2 days). I only selected sentences actually containing taxonomy terms from the [NCBItaxon name dump](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/). This makes the model 5x smaller than word2vec models trained on the entire PubMed database, such as [A survey of word embeddings for clinical text](https://www.sciencedirect.com/science/article/pii/S2590177X19300563) with a strong focus on Taxonomic entities like we wanted. We are not done yet though, since we have __word__ embeddings the question is how we deal with phrases now? Like "blue snake" is two words, not one. There are [several ways to deal with this](), such as taking the average word embedding or the sum. 


What this thought us:
- This indeed is much better at matching semantic similar words, it for example can infer that `Boa constrictor` is a `snake`, and close to `Long Tail Boa`.
- This seems to fail for other trivial examples, such as `snake` vs `snakes`, which is a [known problem](https://www.cs.cmu.edu/~lingwang/papers/emnlp2015.pdf). Using open vocab word representations might solve this but make this much more complex. 
So what about LLMs? Using ChatGPT for big dataset is quite prohobitive, both time and money-wise especially with the hallucinations. Since the LLMs cannot direcly obtain knowledge from the current NCBI taxon database we can train them on the NCBI taxonomy database and ask them questions about this, like they were searching in a document. For example we can use [Facebooks LLAMA index](https://www.llamaindex.ai/) to do this. We give it the ncbi TAXON name dump and it we can ask it for an entry matching `apple`. This somewhat works, but the issue is that again there is no context in the NCBI taxonomy file. It's just taxonomy names with a taxid. Ideally we would give it all PubMed lines and let it index that.. but this takes forever for billions of words. But then LLAMA released [LLAMA 3](https://ai.meta.com/blog/meta-llama-3/), 2 days ago (April 18, 2024) - [try it here](https://www.llama2.ai/). This is very up to date, and incredibly fast. So what if we now change our entire pipeline and let LLAMA 3 do the heavy lifting and we just do the matching of its responses to the database we have exact matching and our word2vec after all. We have a total of `5,713` unique host names. We can do some simple cleaning (weird chars, spaces, removing sp., splitting on commas, etc.) and try exact mapping to the NCBI taxon dump. This already matches `3,324`, leaving us with `2,389` host names. Even by hand these include very complicated mappings. Lets look at a few of them. For example did you know that `c57bl/6j` is `Mus musculus` (see [here](https://www.jax.org/strain/000664))? Or that `bird of prey` should match either `Falconiformes` or `Strigiformes` the two orders of predatory birds ([ref](https://www.britannica.com/animal/bird-of-prey)). Or that `blue sheep` is a different name for `bharal` that should match with `Pseudois nayaur`. That [boxwood](https://en.wikipedia.org/wiki/Boxwood_(disambiguation)) can have different meanings but probably means [Buxus](https://en.wikipedia.org/wiki/Buxus). And what happens if we have host names like `chicken and duck`, do we assign it to chicken or duck? Well LLAMA is smart enough to assign it to [Galliformes](https://en.wikipedia.org/wiki/Galliformes) which includes both. These are a few of the thousands of examples that are very tedious to assign. Of course LLAMA isn't always perfect either, assigning to invalid names such as `unclassified organism` or, by accident, concatenating genus and species such as `mammaliaracatia`. I tried another round of "correct these names" but it often fallsback to just giving the genus name instead of trying to correct the species name. Interestingly it also knowns better for other names, take this `costelytra giveni`, this is absent in the [NCBI taxonomy database](https://www.ncbi.nlm.nih.gov/taxonomy/?term=costelytra+giveni) yet LLAMA correctly says that is a valid species name, probably because of [wikipedia](https://en.wikipedia.org/wiki/Costelytra_giveni). 

So now we can locally run LLAMA3 on our GPUs, see especially [exllama](https://github.com/turboderp/exllama) to make it somewhat fast on our GPUs and ask it to assign it to the closest species. Then we map its results to the NCBI taxonomy dump using a combination of exact matching + our word2vec model. In the end, this leaves us with three categories:
 - EXACT, 100% identical between hostname and NCBI taxo - easy ones and correct
 - CONFIDENT, LLAMA answer that we can directly map to NCBI taxon. LLAMA might be wrong but at least better than nothing 
 - NONMATCHING, LLAMA gives us something but we can't match it, this includes incorrect mappings but also species that are simply absent from NCBI taxonomy, specifically worms or rare animals. 
 - ERRORED, they simply failed to be queried at any step, only 5 in our set.

### Global overview 
We went from `5,713` host names to `3,772` unique taxonomy ids. So still a wide range of taxnomy that we cover. 

Examples of `Homo sapiens` are: `homo sapiens ; human ; human (92 yr old woman) ; human woman ; homo sapiens (adult) ; human (4yr old girl) ; hom sapiens ; human (female 16) ; human (male, 4.5 yrs) ; human child ; gomo sapiens ; homos sapiens ; human body fluids ; patient ; homo sapiens (child) ; homo sapiens; female ; human (male) ; human (infant) ; homme ; human (female) ; homo sapiens (female) ; homosapiens ; hoimo sapiens ; homo sapiens sapiens ; homo sapiens, freshwater ; human male ; homosapien ; homo sapien ; young woman ; homo sapiens (human) ; human (adult) ; human, homo sapiens ; human infant ; human (woman) ; human female ; human (child) ; homo sapiens female ; human (1yr old girl) ; homo sapiens male ; himo sapiens ; homo sapians ; human (neonate) ; 1758) ; homo sapiens; 55 year old ; homo sapiens (respiratory patient) ; homo sapines
`

We do spot a mistake here, `1758)` for some reason LLAMA assigned this to homo sapiens, but this actually [comes from fish gills](https://www.ebi.ac.uk/ena/browser/view/PRJNA788373). 

The second most is `bos taurus`:

`bos taurus ; cattle ; bos bovis ; bos primigenius taurus ; bovine ; ox ; dairy cow ; cow ; veal calf ; cow, bos taurus ; dairy cattle (with subclinical mastitis) ; milking cow ; feedlot cattle ; cows ; holstein friesians ; bostaurus ; holstein cattle ; dairy cattle ; beef ; bos taurus linnaeus ; cow, bos taurus/pig, sus scrofa ; ground beef ; bull ; calf ; bovine (dairy herd) ; beef cattle ; lactating dairy cow ; bos taurus (rumen) ; dairy cows ; food, ground beef ; cattle, bos primigenius ; holstein dairy calf ; brahman steer ; calves ; sausage ; fermented food ; milk ; bovine (dairy cow) ; foetal calf ; holstein ; dairy calf ; bos taurus taurus`

That looks pretty accurate, perhaps except milk and sausage. 

Then we have `sus scrofa`:

`sus scrofa ; swine ; pig ; pigs ; wild boar ; swine (feral) ; pork from farmers market ; pig (foetus) ; pork ; sus scrofa (p53r167h/+) ; diseased pig ; wild pig ; healthy carrier pig ; healthy pig ; sus scrofus ; pig, sus scrofa domesticus ; piglets ; wild boars ; pork chop ; pork from retail store ; sus scrofa (wild boar) ; sus scrofa (wt) ; sus scrofa (pig) ; sow ; feral swine ; pig, sus scrofa ; porcine ; mini pig ; tibetan pig ; zhang yuzhong ; boar ; pig, sus scrofa ; pig (saba) ; sus scrofa (apc1311/+) ; mini-pig ; rpig`

That looks good :)

Then we have `gallus gallus`:
`gallus domesticus ; chicken ; gallus gallus domesticus ; gallus gallus ; chicken breast ; meleagris gallopavo domesticus ; fowl ; bird (fowl) ; broiler ; feral chicken ; gallus gallus breed broiler ; free-range chicken ; gallus gallus domesticuss ; domestic chicken ; chicken (hen) ; broiler chicken ; broiler chiken ; chicken from farmers market ; hen ; laying hen ; egg laying hen ; chicken, gallus gallus ; gallus gallus domesticus isa15 ; young chicken ; birds (fowl) ; brahma chicken ; chicken from supermarket ; qingyuan chicken ; bird (chicken) ; chick ; frozen raw chicken ; gallus gallus (chicken) ; poultry animal (variety unknown) ; 2-week old broiler chicken infected with salmonella enterica serovar heidelberg biosample: samn12082795; sample name: sh-2813-parental 2; sra: srs4982872 ; gallus gallus domesticu`

Here `h-2813-parental 2`  and `sra: srs4982872` seem wrong but they are actually part of a single annotation field that also used `;` as seperator.

### Correcting mistakes 

We now have some pretty good grouping but we also have some obvious mistakes that shouldn't belong in the groups. Going through all groups manually will take long, especially since for many of them we also don't directly know if they belong there, and if not where else they should belong. We can ask LLAMA, given a list of terms and the mapping, if there are any that obviously stand out. 

We first get all matches that aren't `EXACT`, so the matching terms like "Tomato" and then generate a list of everything LLAMA matched with it. This leaves us with `1,314` lists we have to pass to LLAMA. 
For example:
```
❌:  paramecium*sp  ->   waltherarndtia caliculatum
❌:  dipodomys  ->  kangaroo
❌:  scolytus scolytus  ->  bark
❌:  strigomonas galati  ->  insect
❌:  phaeophyceae  ->  seaweed
etc...
```

That looks like a pretty good filtering. Some corrections seems wrong though, for example `tamias  ->  chipmunk` was correct but got removed, so was `phaeophyceae  ->  seaweed`. Given all the corrections we will just ask it to re-assign each of them to an NCBI taxonomy entry once again, only if its confident otherwise NA. This gives us 360 entries that need to be "refined". Lets ask LLAMA to assign all of them to NCBI taxonomy once again, only if 100% sure. This now looks like:
```
[ shepp ]  ->  Ovis aries
[rabbit  ->  Oryctolagus cuniculus
lepus curpaeums]  ->  Lepus europaeus
[lab-strain]  ->  NA
mrigel  ->  NA
male  ->  NA
poultry house  ->  NA
bootsock  ->  NA
urban wild bird  ->  NA
environnment  ->  NA
excluded  ->  NA
hanam-si wastewater treatment  ->  NA
```
Looks decent. So we update the previous table with these corrections but first have to map them to NCBI taxon again. We could remap 160 of them. 

This in total leaves us with 169 that need manual curation, or are simply not valid host names. With the groups:
```
  CONFIDENT  CORRECTION     ERRORED       EXACT NONMATCHING 
       2107         195           5        3312          86 
```

If we now look at the top compressed terms again:

`homo sapiens` with: `
gomo sapiens ; himo sapiens ; hoimo sapiens ; hom sapiens ; homo sapians ; homo sapien ; homo sapiens ; homo sapiens (adult) ; homo sapiens (child) ; homo sapiens (female) ; homo sapiens (human) ; homo sapiens (respiratory patient) ; homo sapiens female ; homo sapiens male ; homo sapiens sapiens ; homo sapiens, freshwater ; homo sapiens; 55 year old ; homo sapiens; female ; homo sapines ; homos sapiens ; homosapien ; homosapiens ; human (1yr old girl) ; human (4yr old girl) ; human (92 yr old woman) ; human (adult) ; human (child) ; human (female 16) ; human (female) ; human (infant) ; human (male) ; human (male, 4.5 yrs) ; human (neonate) ; human (woman) ; human body fluids ; human child ; human female ; human infant ; human male ; human woman ; human, homo sapiens
`

`bos taurus` with: `beef ; beef cattle ; bos taurus ; bos taurus (rumen) ; bos taurus linnaeus ; bos taurus taurus ; bostaurus ; bovine (dairy cow) ; bovine (dairy herd) ; brahman steer ; bull ; calf ; calves ; cattle, bos primigenius ; cow, bos taurus ; cow, bos taurus/pig, sus scrofa ; cows ; dairy calf ; dairy cattle ; dairy cattle (with subclinical mastitis) ; dairy cows ; feedlot cattle ; foetal calf ; food, ground beef ; ground beef ; holstein ; holstein cattle ; holstein dairy calf ; holstein friesians ; lactating dairy cow ; milking cow ; veal calf`

`gallus gallus` with: `2-week old broiler chicken infected with salmonella enterica serovar heidelberg biosample: samn12082795; sample name: sh-2813-parental 2; sra: srs4982872 ; bird (chicken) ; bird (fowl) ; birds (fowl) ; brahma chicken ; broiler ; broiler chicken ; broiler chiken ; chick ; chicken (hen) ; chicken breast ; chicken from farmers market ; chicken from supermarket ; chicken, gallus gallus ; domestic chicken ; egg laying hen ; feral chicken ; fowl ; free-range chicken ; frozen raw chicken ; gallus gallus (chicken) ; gallus gallus breed broiler ; gallus gallus domesticu ; gallus gallus domesticus ; gallus gallus domesticus isa15 ; gallus gallus domesticuss ; hen ; laying hen ; qingyuan chicken ; young chicken`


### Global stats 
We go from `5,708` unique values to `3,769` taxonomy ids. This shows sucesfull mapping of similar names but also exemplifies the wide range of taxa in BV-BRC. 
