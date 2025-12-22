## _DisableInitializers In Upgradeable Contracts

**What I learnt today - 03/12/2025**V

We always see the call `_disableInitializers()` in most upgradeable contract constructors.

The point is simple: even though the initializer function is implemented in the implementation contract, we want a way to ensure that the initialization function can only be called on the proxy contract, not on the implementation itself.

Now, since constructors are called during contract creation, OpenZeppelin provides a small helper function,     `_disableInitializers()`, which updates the initialization state of the implementation contract as “initialized”, without actually calling the initialize function where we add our custom settings.

Because Initializable prevents an initializer from being called more than once, this means the initialize function on the implementation contract cannot be called anymore. At the same time, it can still be called on the proxy, which has its own storage.


