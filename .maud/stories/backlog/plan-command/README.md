What actually runs when the user types the slash command /maud:plan

This is the first step of the maud process. Planning is actually the bulk of the "work" that Maud does so this is rather extensive and has many sub tasks:

- [ ] [Generate .maud/README.md](generate-project-file.md)
- [ ] [Determine how this projects planning should be structured](generate-planning-structure.md)
- [ ] [Run research agents to populate .maud/planning structure with their results](research-agents.md)
- [ ] [Synthesize results of research and the rest of the project files to generate "the plan"](generate-the-plan.md)
- [ ] (optional) [Generate Design](generate-design.md)
- [ ] [Use "the plan" to create stories and populate backlog](generate-stories.md)

Each of these tasks should result in folders or files being created. Then you should present the user with everything you've done, and allow them to review it.

If they want changes, instead of talking to you, the text prompt, they should just edit the files you've generated directly. when those files are in the state they like, they tell you "approved", which moves on to the next step (and commits the change to git)