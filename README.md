Credit where it’s due, this is my take on the work and brain-child of Geoffrey Huntley, I didn’t come up with this stuff, I just tried to make sense of what he’s done =)

This all started for me after reading the Ralph as a Programmer article in The Register. I hopped on over to Geoffrey’s website and got caught up on Ralph and the concept of.. then I took a look around and saw Gas Town. That was intriguing and terrifying, I really like the idea but HELL NO that’s way too deep for my comfort… but there are concepts there that I think I can use.
Thus -
This is my interpretation of the Ralph Loop built on the Gas Town framework. It is intended to be a drop-in ready project agnostic workflow engine capable of taking a concept from brain-dump into a text file to working v1.0, fully hands-off (unless critical).

### What’s in the box?
agent.md – Agent and Subagent behavior rules

projectIdea.md – A blank file to brain dump your idea into. This is the default but any file name can work and be auto-detected.

prompt.md – This is a modified version of Geoffrey’s prompt from Cursed, I used it as a scaffold and modified it work in a more agnostic manner, why re-invent the wheel.

Kb/* - This is the default Knowledge Base for the agents and contains AgentBehavior KB files,  programming and workflow KB files, testing KB files. It will also house any project specific KB files that need to be created. This is a living knowledge base and the General Kbs are intended to transfer from project to project, becoming more expansive and refined with each accomplishment. Currently this contains my General KB’s accumulated over 3.5 projects, you can use and add to them or delete and start fresh with your own data as you see fit, the general and project KB will be created at project creation automatically if empty or missing.
The HR kb is critical to this whole thing working and is REQUIRED for agents to not be toasters, do not delete it. This KB and the HR agent are what provide the behavioral back-pressure to the system and allow it to learn and self-improve.

kb/general/rkp.txt – IF you NEED to give your ai SUDO/ADMINISTRATOR permission, put the password here and ONLY here. FOR THE LOVE OF GODS BE CAREFUL.


HOW TO USE THIS.. THING:
**Script is currently pointed at local files and versioning but can be pointed at Git with ease, just tell the main agent your using git for version control or edit the prompt.md.

1. Put the repo files into the directory you want the project to be created in, DO NOT include any files you don’t want getting sucked up in the process including this readme.

2a. Brain dump your idea into the projectIdea.md file or some other kinda text file, just get the idea out as a starting point.

2b. if you have existing source you want to use, copy it to the a directory named refsrc/ in the project directory (next to kb/). If existing source is referenced in your project idea but you do not have it, the agents will acquire it and place it in the refsrc/ directory. You have been warned.

3. Point your harness of choice at the project directory and tell it to “do prompt.md”.

4. The PM will interview you about the project, this is when you get into the weeds and really hash out what you want to build.

5. After the interview, the agent team will begin the project creation process that will culminate in a fully detailed Software Design Document and the team will wait for instruction. Read it and read it well, this is your last chance to make changes without burning entire sessions worth of tokens to fix an oversight. When you are satisfied with the SDD tell the team to proceed, grab some popcorn and watch the show. This is also when any reference source used goes through the intake process that produces a reference spec, this happens automatically and can really chew a session so keep an eye on your limits.

The team will now finish the project creation process by:
- Creating the Master Spec from the SDD
- Creating the feature specs from the Master Spec
- Creating IOC’s from the feature specs
- Creating the implementation plan (imp_plan.md)
- Creating all other needed support files (session tracking, HR files, directory structures)
- Waiting for instruction to proceed with implementation

Proceeding from here kicks off the full chain of build-test-learn-repeat that will eventually yield a working thing =) This has given me some really good results and I hope it is of use to anyone looking to get into Ai First DevOps.

    Enjoy!!
