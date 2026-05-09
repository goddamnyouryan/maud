Generate the .maud/stories directory. This will always be the same for every project:

.maud/stories
  index.md
  /backlog/
  /current/
  /complete/

Then take all the information from .maud/index.md and .maud/planning/index.md and the designs, if there are any and generate the stories in the backlog.

Then generate the index.md for the stories, which serves as the index for all the stories, as well as the "ui" of the project.

The order the stories are in the backlog is the order that maud will build them in. You are in charge of setting the initial order, but the user can go in and re-prioritize.