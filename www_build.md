<h1>🪙 Umiña Achala Frontend Transition Guide</h1>

<h2><a href="ca://s?q=Mint_NFTs_on_Hedera_testnet">Mint NFTs on CLI</a></h2>
<ul>
  <li>Run your Hedera SDK script with <code>.env.testnet</code>.</li>
  <li>Each mint produces:
    <ul>
      <li>Token ID (e.g. <code>0.0.123456</code>)</li>
      <li>Serial number</li>
      <li>IPFS CID for metadata JSON</li>
    </ul>
  </li>
</ul>

<hr />

<h2> <a href="ca://s?q=Store_minted_NFT_metadata">Store minted metadata</a></h2>
<p>After minting, log results into a registry file or DB.</p>
<p>Example <code>minted_testnet.json</code>:</p>
<pre>
{
  "tokenId": "0.0.123456",
  "serial": 1,
  "cid": "bafybeigdyr...",
  "name": "Umiña Stone #1",
  "owner": "0.0.7890",
  "status": "active",
  "events": [
    {
      "type": "MINT",
      "timestamp": "2026-06-15T16:54:00Z",
      "actor": "0.0.7890"
    }
  ]
}

</pre>

<hr />

<h2><a href="ca://s?q=Frontend_fetch_IPFS_metadata">Fetch registry in frontend</a></h2>
<p>Replace static fetch with:</p>
<pre>
fetch(`${baseUrl}/data/minted_testnet.json`)
  .then(res => res.json())
  .then(items => {
    items.forEach(item => {
      fetch(`https://ipfs.io/ipfs/${item.cid}`)
        .then(res => res.json())
        .then(metadata => {
          // Display metadata in UI
        });
    });
  });
</pre>

<hr />

<h2><a href="ca://s?q=Hedera_mirror_node_API">Use Hedera Mirror Node</a></h2>
<p>Query Hedera’s mirror node REST API for live minted NFTs:</p>
<pre>
fetch(`https://testnet.mirrornode.hedera.com/api/v1/tokens/${tokenId}/nfts`)
  .then(res => res.json())
  .then(data => {
    // Contains minted NFTs + metadata links
  });
</pre>
<p>This removes the need for manual JSON registry. NOTE: Limit to amount of queries</p>

<hr />

## Start with REST polling to keep your local JSON in sync with transfers/burns.

Move to gRPC streaming if you want live updates in the frontend (e.g., “this NFT just changed hands”).

Always reconcile JSON with Mirror Node, since that’s the source of truth for ownership and state.

<h2><a href="ca://s?q=Switch_to_Hedera_mainnet">Switch to mainnet</a></h2>
<ul>
  <li>Update <code>.env</code> to mainnet credentials.</li>
  <li>Frontend fetch logic remains the same.</li>
  <li>Mirror node endpoint changes to mainnet.</li>
</ul>

<hr />

<h2><a href="ca://s?q=WalletConnect_or_MetaMask_for_Hedera">Add wallet integration</a></h2>
<ul>
  <li>Connect via MetaMask or HashPack.</li>
  <li>Users can view their owned NFTs directly.</li>
  <li>Frontend can show both minted items (mirror node) and user holdings.</li>
</ul>

<hr />

## User verification
Add “View on HashScan” buttons that deep‑link to the explorer for each token/NFT.

<p>✅ This workflow keeps your frontend lightweight, while CLI + Hedera APIs handle minting and persistence. It scales from testnet experimentation to mainnet deployment with minimal changes.</p>
