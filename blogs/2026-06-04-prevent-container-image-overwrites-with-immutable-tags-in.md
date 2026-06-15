---
title: "Prevent container image overwrites with immutable tags in Bitbucket Packages"
url: "https://www.atlassian.com/blog/bitbucket/prevent-container-image-overwrites-with-immutable-tags-in-bitbucket-packages"
date: "2026-06-04"
author: "Hamreet Kaur"
feed_url: "https://atlassianblog.wpengine.com/feed"
---
Bitbucket Packages now offers immutable tags, allowing workspace administrators to lock container image tags after initial deployment to prevent unauthorized modifications. The feature supports three configuration levels - protecting no tags, all tags, or specific tags via custom regex patterns - helping teams maintain reproducibility and meet compliance requirements. Once locked, protected tags cannot be overwritten or deleted except by workspace admins.
