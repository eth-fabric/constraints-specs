# Ethereum Commitment API Specifications

**These standards are currently under review / feedback and are not audited.**

This repo contains an open and neutral API specification for proposer commitments on Ethereum. This was developed via a joint effort and contribution from teams across the Ethereum ecosystem who were interested in helping develop and enable proposer commitments on Ethereum. The goal of this effort was to define API standards to enable proposer commitments and subsequently standardize the required coordination across the block construction supply chain.

### Render API Specification
To render spec in browser, you will simply need an HTTP server to load the index.html file in root of the repo.

For example:

```bash
git submodule update --init --recursive # don't forge the submodules!
python3 -m http.server 8080
```

The spec will render at http://localhost:8080.
