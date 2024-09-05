
# How To Own Twenty Percent of the Internet in Ten Guesses

I’ve always had a fascination with combolists. People treat passwords as anything ranging from mundane and disposable to their private confessional. If there’s one piece of information I’d impart to each of you to share with your friends and family, it’s that your passwords aren’t private. I’m looking at you ‘Peeslikeaprincess’.

Going back to the Collection #1–5 series of credential dumps, I’ve been working on leveraging natural language processing (NLP) to try to reverse engineer how people construct passwords. My hypothesis was simple, if I have enough credentials for a person, I should be able to tokenize these passwords to create a grammar to represent how that person thinks about password creation and use that information to attack them.

My process started with the rockyou.txt password list as a proof of concept before going down a rabbit hole and downloading a terabyte of combolists and scraper logs from the dark web. Let me tell you, I’ve never spoken to Troy Hunt, but he really does the lords work because none of this data is ever clean, especially once you get into scraper logs. Storage is the next problem. Over the years, I’ve tried everything from relational databases, to graphs, to document stores, and finally came across a book called “Managing Gigabytes” that made me realize the best and easiest solution would be indexing flat files for permanent storage and optimizing for my hardware. After cleaning, I whittled the terabyte of data down to a little over 4.5 billion unique combos and this is where the fun begins.


## Using Transformer Encoders to Target Individuals
Transformer models were introduced in 2017 which allowed NLP to understand context in the space of sentences and paragraphs by associating words with each other even if they aren’t necessarily next to each other. An example of how to use this would be to compare the strings “he runs fast” and “he’s a fast runner” which some models will tell you are about 80% similar. One may leverage these libraries to compare all the combinations of passwords for a particular user to understand how similar the elements of their password are across multiple observed passwords. A high similarity score indicates it’s unlikely you’d even need to follow my hypothesis on breaking things into grammars because the ordering of characters is similar. This is where understanding how the password requirements of a particular site you’re trying to crack comes into play as this would require the user to make minor changes to their “normal” password to meet these requirements. I found about 25% of users scored highly in terms of password similarity across observed passwords.
Generative AI for Parsing and Hallucination to Determine Entropy

In 2012, [Dan Wheeler at Dropbox released a password strength estimator called ‘zxcvbn’.](https://dropbox.tech/security/zxcvbn-realistic-password-strength-estimation) I can’t tell this story without a nod to his work and the XKCD comic that demonstrates parsing passwords into a grammar so well I’d feel I was plagiarizing by trying to do better. In 2019, when I was buried in the Collection data, this was what I was using to parse passwords. A limitation of zxcvbn is the dictionaries, LLMs are great because someone else has already scraped the internet and tokenized everything into a vector database to give me a dictionary that represents a pinnacle of human collaboration.

Using GenAI to parse passwords worked wonderfully but wasn’t perfect. You see, I was wanting to build a simple grammar so I could write this blog and talk about how there’s only so many original stories in the world and similarly, there’s only so many original passwords. Unfortunately, sometimes LLMs don’t deliver. One of my [favorite cartoons of late](https://www.reddit.com/r/singularity/comments/1b35yo2/people_who_think_something_is_true_because_an_llm/) captures the dangers of LLMs, in that sometimes they will just lie and start making stuff up. In analyzing parsing issues and trying to determine if it was a prompt issue, I noticed that when things started breaking down it was creating extremely complicated grammars. Furthermore, the passwords that weren’t parsing well, tended to have high entropy as you’d expect from an uncracked hash or the output from a password manager. One of the most interesting takeaways from this research, for me, was how to leverage unexpected or undesirable results from LLMs to improve my workflows. By calculating the ratio of tokens in the grammar to the number of bytes in the password, I was able to identify issues with both my data preprocessing as well as users who were likely using password managers. Even failure can be informative.


## A Methodology for Predicting Unseen Passwords
There’s a number of issues in trying to perform this research. First, I had to get a list of email addresses that were old enough to show up in combos, while having some likelihood not to be associated with bot traffic. I found the Twitter leak data from 2021 to be a great resource for sampling users and then mapping to their credentials. Through this research I was largely experimenting by running batches of 10,000 users as this would complete overnight, and the culmination of this research was running against 1,000,000 users randomly selected from this data leak. Across the board, I was seeing just over 40% of sampled users had multiple passwords for me to play with. From here, I had to simulate unseen passwords, so at the beginning of a run I would randomly remove one password from the list of passwords associated with each user and set that as my target value. As I would start testing passwords, I would do so in batches by methodology. When I successfully cracked a password, I’d count the number of guesses at the end of the batch to document the number of guesses to crack a particular user.

As I started assembling the fuzzing engine, I built two functions, one to toggle capitalization of the first character of a token, and the second was to strip special tokens from the last character and iterate through the top 8 specials in use in the dataset !$.@#=*?. The workflow here is to take the set of “seen” passwords, tokenize them into a grammar, toggle capitalization on each of the tokens and add these to the list of tokens, then toggle the specials at the end of each token to add to the set of tokens before brute forcing all permutations of the tokens with attention to sequence.


## Disappointing Results in Many Ways
The goal was never to be able to pwn everyone, but just enough to be significant. I was overthinking things. In testing these functions without tokenizing — literally just passing the passwords in, the horror set in. I was consistently cracking around 17% of users doing nothing more than toggling the capitalization of the first character of every password I’d seen for them. Keep in mind, this 17% is of users I had multiple unique combos for. For my run of 1 million users, this was 17.58% of 404,424 users. For users I wasn’t able to pop by just toggling capitalization of all their compromised credentials, I was able to guess another ~5% by iterating these 8 specials.

Looking back at the section on using transformers to identify similarity, I knew that 25% of users had high similarity across breached or observed passwords and I’d solved 22%. My grand hypothesis was going to be a lot of work for what I estimated would be about 3% improvement which was going to be a lot of work and a lot of processing time to complete for minimal expected improvement. Without further adieu, my final results:

*  1 million email addresses randomly sampled of 200M in 2001 Twitter data leak
*  404,424 users with 2 or more unique passwords
*  17.58% of “unseen” passwords guessed by toggling capitalization
*  22.01% of “unseen” passwords guessed after capitalization and iterating 8 special chars at the end
*  9.25 average number of guesses required for successful cracks
*  7+ days processing on an M1 Max MacBook Pro

[I’ve uploaded my Defcon submission supplement to Github](https://github.com/joshzelonis/FalcoDefconSubmission/blob/main/JoshZelonis-DefconSubmissionSupplement.ipynb), which does not include combo information for anyone but myself and uses a much smaller sample. You may also notice the total guesses is higher than I’m claiming here because I was not checking if capitalization worked before iterating specials in this sample.

## Recommendations
Now that I’ve been down this road, I have three recommendations.
1) IAM providers need to be checking passwords against previously compromised passwords using transformers to make sure they aren’t similar. This can be done before they are hashed and written to metal.
2) The companies relying on password authentication need to push passwordless or at least multifactor authentication.
3) The rest of us need to be using password generators.

