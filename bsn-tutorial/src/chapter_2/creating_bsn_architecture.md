# Creating BSN Architecture

Now that we have a fairly good understanding of how to use BSN, we can now dive into how to design your codebase so it doesn't shoot you in the foot.

In the following pages, I am going to lay out my approach to this with the following feature-set:

1. Reusable templates for quick composition of Scenes
2. Dynamic injection of data to configure scenes 
3. Ability to Save and Load Scenes from JSON Files

This feature-set sets us up to have a production-ready codebase that is built to take on many different kinds of objects with minimal effort.