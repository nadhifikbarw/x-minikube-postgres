## On Secret

In production, you need need to think more holistically on how secrets management
should be carried out, secret management involves multiple productivity and
security concerns, simply deferring such concerns per deployment-basis likely
going to impose hurdles as your services grow.

If your current organization hasn't established any workflow around this
or you're the one responsible to solve this concerns
you could take a look at something such as:
AWS Secret Manager or https://developer.hashicorp.com/vault

Good keyword to help understanding around this process is `ClickOps vs GitOps`
