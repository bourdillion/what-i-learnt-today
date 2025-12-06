## supportsInterface Function

**What I learnt today - 04/12/2025**

The supportsInterface function is part of the ERC-165 standard, which provides a way for contracts to publicly declare which interfaces they implement. It is important because Ethereum contracts don't have built-in type information, there's no way to ask a contract "what can you do?" without this function.

When a contract implements supportsInterface, it returns true for interface IDs it supports and false for those it doesn't. The interface ID is calculated as the XOR of all function selectors in that interface. This allows other contracts or off-chain systems to query: "Does this contract implement ERC-721?" or "Does this vault support ERC-7540?" before attempting to call those functions.

Protocols implement this for several reasons. 
1. it enables safer integrations, you can check if a contract supports a specific standard before interacting with it, avoiding reverts from calling non-existent functions. 
 
2. It's required by many standards (ERC-721, ERC-1155, etc.) as part of their specification, so if you want to be fully compliant with some ERC's, you need to include it 

3. Lastly, it helps with contract discovery and categorization in indexers and explorers like etherscan or basescan for example, you can identify all vaults that implement ERC-7575 by checking their supportsInterface responses.

The main limitation is that supportsInterface is self-reported, so a malicious contract can claim to support an interface but implement it incorrectly or don't even implement it at all. It is just a declaration of intent and not like a guarantee.
