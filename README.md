## Ralph-DevOps
Credit where it’s due, this is my take on the work and brain-child of Geoffrey Huntley, I didn’t come up with this stuff, I just tried to make sense of what he’s done =)

This is my interpretation of the Ralph Loop built on the Gas Town framework. It is intended to be a drop-in ready project agnostic workflow engine capable of taking a concept from brain-dump into a text file to working v1.0, fully hands-off (unless critical).

Let me explain a bit of what I mean by Ralph with a Gas Town framework.. Ralph is in my view a monolithic process, multi-step between Plan and Build yes but the bulk of the the work is the looping prompt.md and the agents.md files. The agents file sets the definitions for the project and front to back rules for the development process.
Gas Town, and this is just my understanding so it may be incomplete/not fully correct, is an orchestration engine for running many Ralphs at once. It is microcosm if unique entities, with layers on layers of controlling agents keeping the whole madhouse in check. It is a beautiful and terrifying thing to behold, well beyond my depth. But it sparked an idea, what would it look like if I invert the structure and put the orchestration engine inside of the Ralph loop and orchestrate the development process and through that the subagents themselves.

So I set about creating this Ralph with the view of a software development company and the different departments and rolls needed for one to function. Each subagent has a job description and a roll to play in the process, and no one has to know everything about everything. Break up the skills and context into bite size bits to maximize focus. This gives a ‘team’ of subagents who carry out the tasks of their departments under the watch of the main agent, to achieve a single goal. The HR subagent (maybe I should change it to AR?) and Helpdesk subagent provide a mechanism for the subagents to learn, remember their learning's and self correct/improve their behavior as they go through the trial and error process of creating working software. In essence a form of file based context storage. This in theory lets the main agent context compact without affecting the working context of the subagents. I say in theory because I have not been able to test this for more than an hour at a time due to rate limits. I have seen main agent compaction happen on several occasions (only once during any given session) and not seen any negative outcomes in the code or process, but this may just be a case of not compacting enough to show context rot if its is a cumulative effect. Currently this is setup for a single agent pair per task (HR is always on watch for patterns) due to the very real dollar costs of running this, but it isn’t hard to switch it from single agent/department per loop to dozens or hundreds per loop.

### What’s in the box?
agent.md – Agent and Subagent behavior rules. This is the brain of the system, give it a read for a complete picture of how all this fits together.

projectIdea.md – A blank file to brain dump your idea into. This is the default but any file name can work and be auto-detected.

prompt.md – This is a modified version of Geoffrey’s prompt from Cursed, I used it as a scaffold and modified it work in a more agnostic manner, why re-invent the wheel.

kb/* - This is the Knowledge Base directory for the agents and contains AgentBehavior KB files, programming and workflow KB files, testing KB files. It will also house any project specific KB files that need to be created. This is a living knowledge base and the General Kbs are intended to transfer from project to project, becoming more expansive and refined with each accomplishment. Currently this contains my General KB’s accumulated over 3.5 projects, you can use and add to them or delete and start fresh with your own data as you see fit, the general and project KB will be created at project creation automatically if empty or missing.

The HR KB is critical to this whole thing working and is REQUIRED for agents to not be toasters, do not delete it. This KB and the HR agent are what provide the behavioral back-pressure to the system and allow it to learn and self-improve.

kb/general/rkp.txt – IF you NEED to give your Ai SUDO/ADMINISTRATOR permission, put the password here and ONLY here. FOR THE LOVE OF GODS BE CAREFUL. This is handing the keys to the castle to the robot and carries a significant non-zero risk that the robot will wreck your castle. Do your homework and understand the dangers before cutting an Ai loose with elevated privileges.


### HOW TO USE THIS.. THING:
**Script is currently pointed at local files and versioning but can be pointed at Git with ease, just tell the main agent your using git for version control or edit the prompt.md.

1. Put the repo files into the directory you want the project to be created in, DO NOT include any files you don’t want getting sucked up in the process including this readme.

2a. Brain dump your idea into the projectIdea.md file or some other kinda text file, just get the idea out as a starting point.

2b. if you have existing source you want to use as a reference, create directory named refsrc/ in the project directory (next to kb/) and put the source folder(s) in there. If existing source is referenced in your project idea but you do not have it, the agents will acquire it and place it in the refsrc/ directory. If you are continuing with an existing project and are modifying existing source create a src/ directory in the main project folder and put the source you are working on in there. 

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
