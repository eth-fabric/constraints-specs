# Ethereum Commitment API Specifications

**These standards are currently under review / feedback and are not audited.**

This repo contains an open and neutral API specification for proposer commitments on Ethereum. This was developed via collaboration and contribution from teams across the Ethereum ecosystem who were interested in helping develop and enable proposer commitments on Ethereum. These efforts started at zuBerlin, continued through Edge City Sequencing Week, and have progresed through community calls (recordings can be found in the pm repo). 

The goal of this effort was to define API standards to enable proposer commitments and subsequently standardize the required coordination across the block construction supply chain.

The spec is largely an extension of the PBS pipeline (see high-level flow below). The specs currently do not have robust documentation, but once solidified, docuemtation will be drafted / published. With that said, the details are in the code / code comments found in this repo. 

![image](https://github.com/user-attachments/assets/0b402fa0-bade-429f-b8cd-fcbd06adc572)

### Render API Specification
To render spec in browser, you will simply need an HTTP server to load the index.html file in root of the repo.

For example:

```bash
git submodule update --init --recursive # don't forge the submodules!
python3 -m http.server 8080
```

The spec will render at http://localhost:8080.
