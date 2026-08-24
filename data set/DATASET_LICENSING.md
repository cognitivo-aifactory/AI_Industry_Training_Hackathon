# Dataset licensing review

Review date: 24 August 2026

This document records the repository-level licensing review requested as part of the AFR corpus remediation. It is an operational record, not legal advice.

## AFR news corpus

**Status: removed from public distribution.**

The full-text AFR articles may be subject to third-party copyright and may contain personal information. The corpus is no longer included in the public repository. See [AFR/README.md](AFR/README.md) for the controlled-access process and Git-history decision.

## Reserve Bank of Australia cash-rate data

**Status: permitted with conditions and attribution.**

The repository contains numerical cash-rate data. The RBA's Copyright and Disclaimer Notice permits use, reproduction, publication, and public communication of the Cash Rate and Cash Rate Materials subject to its conditions. Use must not imply RBA endorsement, must not be unlawful or improperly commercially exploit the data, and must include the attribution required by the RBA.

Required attribution for this repository:

> Source: Reserve Bank of Australia 2026

Official terms: <https://www.rba.gov.au/copyright/>

## ASX company-price data

**Status: redistribution rights not yet confirmed.**

The repository contains OHLCV records with `.AX` ticker symbols, but the files do not identify their source, acquisition method, licence, or required attribution. The filename and data shape are not sufficient evidence of redistribution rights.

ASX warns that some market data on its services is supplied by third parties and may not be republished or redistributed without permission. Before treating these files as cleared, the data owner must record:

1. the original provider and download method;
2. the terms or licence that applied when the data was obtained;
3. whether public redistribution is permitted; and
4. any attribution or notice required in this repository.

Official references:

- <https://www.asx.com.au/legals/data-disclaimers>
- <https://www.asx.com.au/legals/terms-of-use.html>

Until provenance and permission are documented, do not describe the ASX dataset as licence-cleared.

