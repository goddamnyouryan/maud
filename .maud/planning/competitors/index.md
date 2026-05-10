Maud is directly inspired by the claude code plugin GSD, but it functions closer to how the product development loop has worked for Ryan MacInnes, who has been a freelancer / startup technical founder for going on 20 years now.

It takes the following inspiration from GSD: Local context management, agent spawning with scoped context, conversation-to-disk persistence, resumability.

It differs from GSD in a number of important ways:

GSD is too rigid: arbitrary phase counts, sometimes too much planning, sometimes too little, because it has hard coded numbers. Doesn't think about the project holistically, but discusses and plans each phase in a vacuum. Research bakes all these assumptions in to the codebase that never get surfaced to the user. Generates meaningless almost unreadable to human boilerplate with tons of duplicate information. Creeps scope to a crazy degree, makes tons of premature optimizations.

Maud is meant to avoid all of these issues.