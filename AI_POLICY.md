# AI Policy

## Principles

- **AI usage should be disclosed.**

- **The human developer is the primary author**, not just nominally but in practice. AI is a tool like any other development convenience, not as an unsupervised agent.

- **The developer must fully understand all code** and be capable of having written it without AI help, having reviewed every line for its purpose and impact.

- **Discussions and documentation may be drafted with AI assistance, but must be reviewed and edited by the developer before submission.**

- **No AI-generated media (art, images, video, audio, etc.), even as placeholders.** Text and code are the only acceptable AI-generated content.

Contributions using AI tools are welcome on the same terms: you've reviewed and understood every line, you can explain it, and you disclose the assistance in the PR.

## Explanation

I use Anthropic's _Claude_ (mainly via _Claude Code_) for researching problems and approaches, and more heavily for the tedious parts: boilerplate, release engineering, and keeping documentation up to date. Architecture, design and core logic are mine. Writing Minecraft mods is a hobby for me, rather than my day job, so I'm not going to waste the little time I have on dull boilerplate that a good AI can do perfectly well and with fewer mistakes. (In my professional capacity, I have different criteria!)

Modern AIs like _Claude Opus_ and _Fable_ are surprisingly capable of designing entire systems, but they usually miss the simple solutions, compromises and adjustments that lead to simpler, more reliable, more consistent results. Whenever I've relied too much on AI for the big-picture work, the result has been technically adequate but often overcomplicated. This is where human judgement still comes out on top... as long as the human in question is competent and experienced in the field, that is!

In principle, I have no problem with AI as a tool, much as autocomplete, IDEs and version control revolutionised software development by automating what used to be done by hand. What I'm _not_ comfortable with is _vibe coding_, where the "developer" is detached from the product and relies on AI to do work they couldn't do themselves. AI belongs on mechanical tasks, with the human in full control and understanding every step.

So where code is predominantly AI-generated, I review every line and often rework it, and it _must_ be tested carefully by a human before release. I'm responsible for everything in this repository, whoever or whatever typed it first. Bugs and design flaws are mine.

And, for the code I have written myself, I use AI as a checker to find bugs, flaws and possible improvements in my efforts. In effect, AI and I check each other.

Anyway, that's my stance on the matter. I respect that others have different opinions for pragmatic, economic, ethical and/or ideological reasons and even boycott projects that have used AI, and they're free to do so. However, I choose to cautiously embrace AI for limited uses, and I'd hope that my position would be given the same respect.  Since my projects are very liberally licensed, you're free to recreate them without AI if you wish, and I'd be happy to consider such contributions if submitted.

Oh, and incidentally, I imagine you may be thinking this oververbose document is a sign of generative AI at work, but no, it's the sign of neurodivergent hyperverbosity. If anything I tend to use AI to try to distill and reduce my word count — and I've been (over)using em-dashes and semicolons long before LLMs were invented!

-- Tom Gidden, 22 September 2026