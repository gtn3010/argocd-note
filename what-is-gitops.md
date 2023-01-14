1) A desired state as expressed in a declarative system

This is about the ‘single source of truth’ – essentially, keeping a description of the entire system in one place, so you’ve got something to compare the production version against.

As Weaveworks’ Brice Fernandes puts it, "that means a system that's managed by GitOps must have its desired state completely expressed declaratively as data and that should be readable and writable by both machines and humans."
2) Immutable versions of that desired state

You can’t rewrite history. Everything gets recorded and nobody can go back and make changes to previous versions of the system. This way, you know that if (or, rather, when) you need to roll back, you’re rolling back to something that definitely worked. In Brice’s words, “versions should be immutable, and you should retain a complete version of history so that there's no history rewrites.” The group went on to discuss how this really gets to the heart of what GitOps is all about.
3) Continuous state reconciliation

GitOps isn’t just about Git and Kubernetes. It relies on additional software agents to monitor what’s running in production, checking for any drift from the version described in Git. If they don’t like what they see, that’s when they start to fire off alerts.

    “The fact that the system state is automatically maintained by the reconciliation agents to reduce system draft is something that we've been doing within various aspects of Microsoft for quite a while. And the fact that it really helps with developer productivity. Developers can focus on code rather than, you know, the actual, the systems themselves and also the security aspect.”
     – Chris Sanders, Program Manager, Microsoft

4) Operations through declaration

The last principle the group agreed on was the idea that in GitOps, absolutely all operations actions are performed declaratively – meaning that nothing can be done unless it is done through Git. Without this, the integrity of the system would be compromised.

As Cornelia Davis summarises, “Git is the interface to operations. The principle of operations through declaration says, that's how you do ops. You don't go into clicky, clicky interfaces, or scripts, or some other APIs. It is through changes to that declarative state.”

Or as Jesse Butler, Senior Developer Advocate, Kubernetes team AWS puts it, “How do you get to the ops part? And that's really, to me, the fundamental change over the last couple of years... we have the automation integrated with the deployment environment. That's the ideal implementation. So now we're looking at something internal inside of the distributed system, inside the complex system, which is actually monitoring for changes to itself. Pulling those changes and applying those changes based on the desired state, which is in version control, which is immutable. So this is really the sort of paradigm shift for me.”

ref:
https://www.weave.works/blog/opengitops-the-vendor-neutral-gitops-project
