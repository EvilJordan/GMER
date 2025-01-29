Posting this in #general for feedback!

So what I'm trying to do is design our NFTs in a way that extends beyond just simple collectibles, preparing for actual utility, interoperability, and upgradeability _in the future_.

The problem is metadata can either be **mutable** or **immutable**, but not both, and anything we want to do should be compatible with existing standards, and use the `tokenURI()` method. There are no current standards or adopted paradigms to handle this, and GameStop is in a position to influence that.

A note before reading: our `tokenID` is made up of a deterministic formula that can be decomposed into: `companyID`, `collectionID`, and `editionID`. `myCompanyID` is equivalent to `companyID` unless otherwise specified in a function call. Additionally, Brecht at Loopring and I worked this out over the course of the past few days, so at least there is one intelligent person with eyes on this already.

The crux of this proposal is what I'm calling the **G**ameStop **M**etadata **E**(x)ternal **R**egistry – **GMER**.

This is a L1 contract that exists sort of like ENS in that it holds records and pointers to data elsewhere.

Anyone could register a "top level" `companyID` within GMER, though we would reserve a large chunk for GameStop to own and manage. A `companyID` is registered to the `msg.sender` that created it in a mapping of `_companyID -> address`, though this could be extended to allow/revoke permissions for other addresses as well. Only `companyID` owners/managers are allowed to create `collectionID`s underneath that `companyID`. There is now one more level, by default, with the same `myCompanyID` again. This indicates that this `Company.Collection.Company` is the source of truth for _immutable_ data relating to the individual `editionIDs` that fall under this Collection. `myCompanyID` is an array, with the first index - called a `patch` containing a key of `directory`, pointing to the immutable IPFS directory hash we create during our GameStop minting process. The structure thus-far looks like this:

```
100 // companyID
	001 // collectionID
		100 // myCompanyID (companyID again)
			0 // first patch
				directory: ipfs://abc // this is our "destination"
```
```
{"100":{"001":{"100":[{"directory":"ipfs://abc"}]}}}
```

The GMER contract needs a write method to allow _anyone_ to append at the `companyID/collectionID/*` level. These would be "level 2" data. Data is appended with a setter method: `setMetadata(tokenID, myCompanyID, directory, destination)` where `myCompanyID` is a constant established on first call, and associated with `msg.sender`. `directory` is a Boolean switch to determine whether to set the `destination` on an `editionID` as parsed from the `tokenID` or at the `directory` key for the `collectionID`. Each write appends a new index to `companyID/collectionID/myCompanyID[n]/directory` or `collectionID/myCompanyID[n]/tokens/editionID`.

The GMER contract needs a read method: `getMetadata(tokenID, myCompanyID, patchNumber)`. If `myCompanyID` is omitted, `myCompanyID` is derived from the `tokenID`. If `patchNumber` is omitted, `patchNumber` is equal to `myCompanyID` index length - 1. If no entry exists for the given `tokens/editionID `(parsed from the `tokenID`), the `directory` key is checked for a generalized `destination`. The method then returns the `destination` stored in whatever space it finds. If no `directory` entry is found for a given `patchNumber`, decrease the `patchNumber` until we have a hit on `tokens/editionID` or `directory` (max number of loops is index length - 1).

`getMetaData()` is also added to the NFT contract as a convenience method, calling the `getMetaData()` method of GMER.

`getMetadata()` is used for returning the (im)mutable metadata for an NFT. Marketplaces, games, and other actors would need to be made aware this NFT contains this functionality.

`tokenURI()` should be set to make a `call()` to GMER's `getMetaData()` method and _always return the value for patchNumber 0_ of the given `tokenID` using the traversal process to check for an individual `editionID` or a `directory`. This locks in vendor "level 1" immutable metadata.

`mint()` can optionally be extended by other contracts wanting to use GMER to add an entry to the registry for the minted token by including additional parameters, overriding its function, and calling `setMetadata()` from the NFT contract.

The problem of cost to call the `setMetadata()` method of GMER seems like a big issue, therefore we recommend that `myCompanyID` create all their metadata, push to IPFS, and then create an IPFS directory. Store that directory hash/destination in the `directory `key of `companyID/collectionID/myCompanyID[n]/directory`. Alternatively, a hash/destination can be stored on an individual `tokens/editionID` level. Missing `destination`s fallback down the tree to an earlier version.

All of this functionality allows for a company to push new metadata to an NFT over time, while maintaining history as an immutable state. This data could describe various attributes, contain instructions for NFT display in a given engine, or who knows what else. The point isn't to define those use-cases now, but allow our NFTs to be special so they can evolve with whatever comes next.
