---
layout: post
title:  "Anatomy of Estonian declension, part I"
date: 2023-01-28 18:48
comments: true
categories: wikidata linguistics
---

What is "declension"? As [per Wikipedia](https://en.wikipedia.org/wiki/Declension): 

> In linguistics, declension (verb: to decline) is the changing of the form of a word, generally to express its syntactic function in the sentence, by way of some inflection. Declensions may apply to nouns, pronouns, adjectives, adverbs, and articles to indicate number (e.g. singular, dual, plural), case (e.g. nominative case, accusative case, genitive case, dative case), gender (e.g. masculine, neuter, feminine), and a number of other grammatical categories.

Estonian language is exactly one of the languages, where declension is quite important. Maybe not so much as in Slavic languages, 
but way more important, compared to English or Dutch. Estonian _names_ (primarily _nouns_ and _adjectives_) count 14 (by some definition - 15) cases,
but most of those are simple _agglutinations_, i.e. appending a certain constant suffix to some basic form. However, *some* cases are 
not so simple, and getting used to them takes quite some time. In this blog post, we'll try to identify some of the ways Estonian declension works. 

Let's go! In fact, I have a database of Estonian lexemes (described in [another blog post](2021-02-01-long-road-to-lexeme.html)), 
and we'll be using it for our research. I will not be using the Wikidata this time, however it must be fully possible to 
reproduce with Wikidata Lexemes as well. 

Let's begin with formally identifying the "real" cases, i.e. the ones where forms are NOT (or not always) produced by 
simply appending a suffix to basic form. In order to that, we'll find the lexemes (words) where inflected form + case syntax 
is different from the actually observed form. 

Let's begin with the usual suspects: "nina taga" group of cases (i.e. forms, ending in `-ni`, `-na`, `-ta` and `-ga`): 

```sql
with base as (
select r.representation, ef.paradigm_id as id
from ekilex_forms ef  
join representations r on ef.word_representation_id = r.id
where ef.form_type_combination_id  = 2 and r.representation <> '-'
),
declined as (
select r.representation, ef.paradigm_id as id
from ekilex_forms ef  
join representations r on ef.word_representation_id = r.id
where ef.form_type_combination_id  = 14 and r.representation <> '-'
)
select base.id, base.representation, declined.representation as inflected, concat(base.representation, 'ga') as suffixed
from base 
join declined on base.id = declined.id
where declined.representation <> concat(base.representation, 'ga');
```

this produces ~70 results, which is basically nothing. In all cases, these can be explained by multiple observed basic forms 
(_basic form in Estonian is `omastav`_, i.e. genitive case). So, obviously, where there is more than one omastav, we'll see "discrepancies".

If we throw away all the words with more than one singular genitive, we'll get just these 4 results: 

| Word   | Base form  | Translation         | 	Actual `-ga` form	 | Formula-based form |
|--------|:-----------|:--------------------|:--------------------|:--------------------|
| keegi  | kellegi    | someone             | 	kellegagi          | kellegiga           |
| kumbki | kummagi    | one of both         | 	kummagagi          | kummagiga           |
| miski  | millegi    | something / nothing | 	millegagi          | millegiga           |
| ükski  | ühegi      | no one/ anyone      | 	ühegagi	           | ühegiga             |

And here we see very interesting phenomenon, remnants of a system, that is still [very much present in Finnish](https://uusikielemme.fi/finnish-grammar/the-order-of-finnish-suffixes): 
relative positioning of suffixes. `-gi/-ki` is so-called _emphatic_ suffix, i.e. you add it to an end of the word when 
you want to put special emphasis, _highlight_ this word in a phrase. And, as we can see, it has to be placed in the last 
position, _after_ the case suffix `-ga`. By the way, words with `-gi/-ki` normally never make it in the dictionary, because, 
it being emphatic, it can be easily applied to any worm of any word. But these particular words, initially being just 
emphatic forms of other words, ended up being very important and deserving own place in the vocabulary. 

Now, when we know of these exceptions, it's safe to exclude them as well and exclude the `-ga` case 
(which is called [comitativ](https://en.wikipedia.org/wiki/Comitative_case#Estonian), and means "with, or including, this word"),
from the consideration as the _real_ inflectional case.

Let's now look at `-ta` case ([abessive](https://en.wikipedia.org/wiki/Abessive_case), which means "without this word"). Same thing! In all of the cases 
(except those we've already excluded), the _observed_ (as provided by our [reference dictionary](www.sonaveeb.ee). 

Same results for `-na` ([essive](https://en.wikipedia.org/wiki/Essive_case)) and `-ni` ([terminative](https://en.wikipedia.org/wiki/Terminative_case)) cases.

We get some interesting results with [translative](https://en.wikipedia.org/wiki/Translative_case) `-ks` case: 

| Word | Base form  | Translation         | 	Actual case form | Formula-based form |
|------|:-----------|:--------------------|:------------------|:--------------------|
| see  | selle      | this | seks              | selleks |
|seesama | sellesama | this same | sekssamaks        | sellesamaks |
|seesama | sellesama | this same | sellekssamaks     | sellesamaks |

These observed discrepancies are in fact again consequences of duplication, but in this case, not of the base form but of the "cased" form. 
The word `see` [has 2 forms](https://sonaveeb.ee/search/unif/dlall/dsall/see/1) for singular translative: `seks/selleks`, 
so we get 1 result where dictionary form is different from the _formula_ of "_base form + **-ks**_". It's even more "interesting" 
with [`seesama`](https://sonaveeb.ee/search/unif/dlall/dsall/seesama/1), which according to dictionary has 2 translative case forms: 
`sellekssamaks` and `sekssamaks`, so this word can be considered one and only _true_ deviation from the _case formula_ 
for the translative case in the whole of Estonian language (in singular only, there's entirely different story with plural). 

Now, let's look at so-called _locative_ cases, that are used (for the most part, but not only) to reflect positioning (placement) 
of object or direction of action. Let's begin with [ablative](https://en.wikipedia.org/wiki/Ablative_case) case that has `-lt` case suffix. 

| Word | Base form  | Translation | Actual case form | Formula-based form |
|------|:-----------|:------------|:-----------------|:-------------------|
| see  | selle      | this	       | selt	            | sellelt |
|seesama	| sellesama  | this same   |	selleltsamalt	| sellesamalt |
|seesama	| sellesama  | this same   |	seltsamalt	|sellesamalt |
|too	| tolle      | that        |	tolt |	tollelt |

Again, usual suspects, which we should probably exclude from further analysis. These words and these forms, apparently, 
occur so often in everyday speech, that they have developed multiple _realizations_, `selt/sellelt`, and, by analogy, 
`tolt/tollelt`, all of which have gained enough traction to be included in the dictionary.   

Next one: [adessive](https://en.wikipedia.org/wiki/Adessive_case) case, with an `-l` suffix. Having excluded words from 
above tables, we see these new forms popping up in _naughty forms list_: 

| Word | Base form | Translation | Actual case form | Formula-based form |
|------|:----------|:------------|:-----------------|:-------------------|
| kes  | kelle | who        | kel              | 	kellel |
| mis | mille | what | 	mil |	millel |

Same story as above, but also there's some lesson lurking there: we observe that a form with `-llel-` is very frequently 
reduced to just `-l`: `millel/mil`, `sellelt/selt`, `kellel/kel`, `tollelt/tolt`, and so on. This is probably our first 
hint on existing of two parallel paradigms (spoiler: the longer one, and the shorter one) for most of the personal pronouns, on which we probably will discover more 
going forward.

Next one: [allative](https://en.wikipedia.org/wiki/Allative_case) case with `-le` ending. 

| Word | Base form | Translation    | Actual case form | Formula-based form |
|------|:----------|:---------------|:-----------------|:-------------------|
| ma |	mu	| I              | mulle            | 	mule              |
| sa |	su | you (informal) | 	sulle           | 	sule              |
| ta |	ta	| he/she |  talle           | 	tale              |

These are again, three instances of _shortened_ paradigm for personal pronouns, and we can observe that `-le` in these 
words is applied as `-lle`, and I thus far have no idea as of "why". I guess it just sounds more natural this way, to a native speaker!

With this, we have covered so-called "exterior" group of locative cases (the ones that might  be translated using "on" or "onto").  
Next, we'll cover "interior" ones (that might be translated with "in" or "into"). 

First, [elative](https://en.wikipedia.org/wiki/Elative_case) case, with the `-st` formula. And here, we observe one exception, 
that occurs in many words, but all of these words are just compounds with `kodu` as basic component:

| Word | Base form | Translation   | Actual case form | Formula-based form |
|------|:----------|:--------------|:-----------------|:-------------------|
|isakodu |	isakodu | father's home | isakodunt        |	isakodust |
|kodu |	kodu | 	 home        | kodunt           |	kodust | 
|koolkodu |	koolkodu | school house  |	koolkodunt|	koolkodust |

So, one more exception in our collection! Not sure how it is explained though

Next, [inessive](https://en.wikipedia.org/wiki/Inessive_case), with `-s` formula. No exception! Phew...

And, final locative case: [illative](https://en.wikipedia.org/wiki/Illative_case), with `-sse` formula.
No exception again! Wait a minute, this is... fishy. I know for a fact that in this case there's tons of words that do not comply with a formula. 
Let's check out some paradigm, for example that very same `kodu`, [on Sõnaveeb](https://sonaveeb.ee/search/unif/dlall/dsall/kodu/1). 
Aha! Here, we see what has happened: Sonaveeb (which a website, maintained by [Eesti Keele Instituut](https://portaal.eki.ee/), i.e. 
Institute of Estonian Language), simply considers all such "exceptions" to be instances of a separate case, so-called "lühike sisseütlev", 
i.e. "short illative", this is where it diverges from the [English Wikipedia on the subject](https://en.wikipedia.org/wiki/Estonian_grammar#Nouns) (or, rather, wikipedia diverges a bit).

Anyway, this "short illative case" does not have a formula. This way, we have established that Estonian language has 4 "real" 
cases, at least in singular: nominative, genitive, partitive, and "short illative". All the rest can be, by and large, 
produced with a simple formula (minus some exceptions). 

If we look at the plurals, it is for the most part the same situation: 

| Case        | Suffix | Number of exceptions |
|-------------|:-------|:---------------------|
| Comitative  | -ga    | 0                    |
| Abessive    | -ta    | 95                   |
| Essive      | -na    | 0                    |
| Terminative | -ni    | 148                  |
| Translative | -ks    | 74 556               |

Wow, there clearly is something going on there! Let's have a look at those massive discrepancies for a 
_plural translative_ case:

| Word | Base form  | Translation      | Actual case form | Formula-based form |
|------|:-----------|:-----------------|:-----------------|:-------------------|
|aabits| aabitsate	 | alphabet         | aabitsaiks       | aabitsateks        |	
|äädikas| äädikate  | vinegar          | äädikaiks	       | äädikateks         |
|aadel|aadlite| nobility| aadleiks         |aadliteks|

and so on... 

So, what we're dealing with here is the so-called _ghost form_: `aabitsai`/ `äädikai` /`aadlei` which is not given in
dictionaries, but is used as a basic form for _some_ words for forming _some of_ the cases for _plural_ number. 
Now, how prevalent is it? Well, in my database, there's `85,261` _name_ paradigms, and `74,556` of those seem to use this 
`plural root` shadow form (for _translative_ at least), which is ~87%.

Let's see how many discrepancies other cases have:

| Case        | Suffix | Number of exceptions |
|-------------|--------|----------------------|
| Comitative  | -ga    | 0                    |
| Abessive    | -ta    | 95                   |
| Essive      | -na    | 0                    |
| Terminative | -ni    | 148                  |
| Translative | -ks    | 74 556               |
| Ablative    | -lt    | 74 556               |
| Adessive    | -l     | 74 556               |                     |
| Allative    | -le    | 74 556               |
| Elative     | -st    | 74 556               |
| Inessive    | -s     | 74 556               |
| Illative    | -sse   | 74 556               |

Surprising, even suspicious, uniformity. Let's have a look at, say, _translative_ case forms and try to extract a `plural root` form out of those. 

N days later...

Now... once we have identified the mysterious, shadow "root plural" form, let's build a list of discrepancies while taking also these forms into account. 

What do we have for _translative case_: 0. Cool

Let's have a look at _(plural) ablative_ case now. 0 discrepancies again. 

Ok, let's compose a table now, with shadow "root-plural" form included as a base for producing suffixed forms: 

| Case        | Suffix | Number of exceptions |
|-------------|--------|----------------------|
| Translative | -ks    | 0                    |
| Ablative    | -lt    | 0                    |
| Adessive    | -l     | 0                    |                     |
| Allative    | -le    | 0                    |
| Elative     | -st    | 0                    |
| Inessive    | -s     | 0                    |
| Illative    | -sse   | 0                    |

So, basically, (almost) all the discrepancies we've seen above, for the plural forms, can be explained by "root plural" form, 
which is defined on ~87% of all lexemes.

Now let's get back to the small number of discrepancies for _abessive_ and _terminative_ cases that we have seen above.

For _abessive_ case these are: 

|nominal|base|case|inflected|suffixed|
|-------|----|----|---------|--------|
|ebajalg|ebajalge|genitive|ebajaluta|ebajalgeta|
|ebajalg|ebajalgade|genitive|ebajaluta|ebajalgadeta|
|eesjalg|eesjalge|genitive|eesjaluta|eesjalgeta|
|eesjalg|eesjalgade|genitive|eesjaluta|eesjalgadeta|
|esijalg|esijalge|genitive|esijaluta|esijalgeta|
|esijalg|esijalgade|genitive|esijaluta|esijalgadeta|

... and so on, but all of these are for lexemes, that have `jalg` (leg) as it's root. And we can see that `*jalu` is the shadow 
"root plural" form for these words. So, the exception to memorize here is that `Xjalg` also uses shadow "root plural" 
form for this case, unlike the other words, that only use `plural genitive` form for this case.

And we've also seen discrepancies for the _terminative_ case, let's look at those:

|nominal| meaning | base    |case|inflected|suffixed|
|-------|---------|---------|----|---------|--------|
|kõrv| basket  | kõrvade | genitive |kõrvuni|kõrvadeni|
|põlv| knee    | põlvede | genitive |põlvini|põlvedeni|
|rind| breast  | rindade | genitive |rinnuni|rindadeni|
|rind| breast  | rinde   | genitive |rinnuni|rindeni|
|silm| eye     | silmade | genitive |silmini|silmadeni|
|silm| eye     | silme   | genitive |silmini|silmeni|

And a lot of other words, all based on one of these roots. So, another exception to remember is that words, based on one 
of these roots, also use it's shadow "root plural" form to build a _terminative_ case.   

Let's also quickly confirm probably the first rule one ever learns about Estonian morphology: 

> _nominative plural_ = _genitive singular_ + `-d`

My teacher even used to say:
> Don't memorize the _second form_ (a.k.a _the genitive_), memorize the plural instead! You'll get two for the price of one.

Oh wow, there are some exceptions indeed. But all of them are _pronouns_, which are irregular anyway, and declension of 
pronouns just has to be remembered. 

And finally, let's summarize what we have learned about oh-so-scary Estonian case system: 

## Declension of names (no pronouns)


| Case           | Singular                                                                                               | Plural                                                                                                                                 |
|----------------|--------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Nominative     | the word as in dictionary                                                                              | _singular genitive_ + `d`                                                                                                              |
| Genitive       | has to be memorized                                                                                    | has to be memorized                                                                                                                    |
| Partitive      | has to be memorized                                                                                    | has to be memorized                                                                                                                    |
| Short illative | has to be memorized                                                                                    | has to be memorized                                                                                                                    |
| Illative       | _singular genitive_ + `-sse`                                                                           | _plural genitive_ + `-sse` <br/> or _root plural_ + `-sse`                                                                             |
| Inessive       | _singular genitive_ + `-s`                                                                             | _plural genitive_ + `-s` <br/> or _root plural_ + `-s`                                                                                 |
| Elative        | _singular genitive_ + `-st` <br/> *except for words, ending with `kodu` - these will have `-nt` ending | _plural genitive_ + `-st` <br/> or _root plural_ + `-st`                                                                               |
| Allative       | _singular genitive_ + `-le`                                                                            | _plural genitive_ + `-le` <br/> or _root plural_ + `-le`                                                                               |
| Adessive       | _singular genitive_ + `-l`                                                                             | _plural genitive_ + `-l` <br/> or _root plural_ + `-l`                                                                                 |
| Ablative       | _singular genitive_ + `-lt`                                                                            | _plural genitive_ + `-lt` <br/> or _root plural_ + `-lt`                                                                               |
| Translative    | _singular genitive_ + `-ks`                                                                            | _plural genitive_ + `-ks` <br/> or _root plural_ + `-ks`                                                                               |
| Terminative    | _singular genitive_ + `-ni`                                                                            | _plural genitive_ + `-ni` <br/> *except for words based on `silm`, `põlv`, `kõrv`, `rind` - these will also have _root plural_ + `-ni` |
| Essive         | _singular genitive_ + `-na`                                                                            | _plural genitive_ + `-na` <br/>                                                                                                        |
| Abessive       | _singular genitive_ + `-ta`                                                                            | _plural genitive_ + `-ta` <br/> *except for words based on root `jalg` - these will also have _root plural_ + `-ta`                    |
| Comitative     | _singular genitive_ + `-ga`                                                                            | _plural genitive_ + `-ga` <br/>                                                                                                        |


So, as we can see: out of 30 possible case forms, only 8 have to be memorized, all the rest can be easily produced by 
simply adding a well known suffix to one of three basic forms. And there's just 6 roots, that produce certain exceptions.   

I hope this post helps de-mystify the subject and bring down your fear of learning Estonian!! 

Palju õnne, ja ruttu nägemiseni! (which means "Good luck, and see you soon!")