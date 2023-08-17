# Bellatrix -- Honest Validator

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Validator registration](#validator-registration)
  - [Preparing a registration](#preparing-a-registration)
  - [Signing and submitting a registration](#signing-and-submitting-a-registration)
  - [Registration dissemination](#registration-dissemination)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Validator registration

### Preparing a registration

To do this, the validator client assembles a [`ValidatorRegistrationV2`][validator-registration-v2] with the following
information:

Fields added to `ValidatorRegistrationV2` which was not there in `ValidatorRegistrationV1`

* `proposer_builder_commitment`: the type of block the proposer wants the builder to build.

### Signing and submitting a registration

The validator takes the constructed `ValidatorRegistrationV2` `message` and signs according to the method given in
the [Builder spec][builder-spec] to make a `signature`.

This `signature` is placed along with the `message` into a `SignedValidatorRegistrationV2` and submitted to a connected
beacon node using the [`registerValidator`][register-validator-api] endpoint of the standard validator
[beacon node APIs][beacon-node-apis].

Validators **should** submit valid registrations well ahead of any potential beacon chain proposal duties to ensure
their building preferences are widely available in the external builder network.


