## Self-Destruct is being deprecated
**what i learnt today - 05/12/2025**

As of EIP-6780 (activated in the Cancun upgrade in March 2024), selfdestruct no longer actually destroys contracts or refunds gas in most cases. It now only sends the contract's ETH balance to a specified address, but the contract code and storage remain on the blockchain.
This was a massive change because selfdestruct was previously used for upgrade patterns and cleaning up temporary contracts. The change was made for security reasons - selfdestruct created weird edge cases where contracts could be killed mid-transaction, and it made Ethereum harder to scale.
So if you see old tutorials mentioning selfdestruct for upgradeable contracts or gas refunds, that pattern is now obsolete.
