# Proposed Framework for a Secure Private Direct Message System

## Desired Attributes of a Secure and Private Direct Message System
- Messages are encrypted and only readable by the intended recipients
- Messages are routed with as few intermidiaries as possible
- Messages are short lived within the message system and never persisted beyond a TTL
- There is no public or discoverable link between senders and receivers
- There is minimal latency in message routing
- The system is resistant to spam and other abuse
- The system leverages existing accepted protocols
- There is no significant cost to users
- The system is widely available to anybody with internet access

## Abstracted Participants (Humans, Software, Devices)
- Message: A Direct Message (DM) consists of text or data to be privately conveyed from a sender to a receiver
- Sender: A person or process that composes and sends a message
- Sending Client: Software used by the sender to send a message
- Message Relay: Software/hardware used to move and/or store a message on the open internet
- Notifier: A participant used by a sender to tell a receiver and/or receiving client that there's a message for them
- Receiving Client: Software used by the receiver to fetch and read a message
- Receiver: A person or process that consumes a message

## Using Nostr and the Bitcoin Lightning Network
In this proposal
- Sending and receiving clients are specialized Nostr clients or that part of a general Nostr client that deals with Direct Messages.
- Message relays are Nostr relays.
- The notifier is a payment on the lightning network.

### High Level
- Sending client uses a one-time key pair for each outgoing message.
- The encrypted message is sent in a nostr event.
- The recipeint is never mentioned in the nostr event.
- The message sender also sends a lightning payment to the recipient:
  - The receiving client uses an encrypted comment in the payment to fetch the DM from a Nostr relay
  - The lightning payment transmital (message notification) is completely outside of the Nostr arhitecture
 
### Note about Nostr Relay Involvement
There are two approaches to publishing the DM event:
1) Publish to one or a small number of relays
   - Sender could include a lightning payment to the relay
   - Relay doesn't need to be on either party's regular relay list (but they need relevant write and read access)
   - Minimizes visibility that a DM was ever sent
2) Use Blastr to send it everywhere
   - "Blastr" is used generiallcy here to mean a large number of relays.
   - A potentially large number of clients will pull the event making it difficult or impossible to pick up on who can decrypt it.
   - Makes lightning payment to relays impractical

# How it Works

## Sender
Enters "Hello Bob" into their DM client and hits "Send."

## Sending client
1) Generates new nsec/npub pair (onetime_nsec/onetime_npub) for sending a nostr event.
2) Creates an identity_token by signing the onetime_npub with nsec for senders_known_npub.
3) Encrypts message using receivers_known_npub.
4) Determines if event will be published with Blastr, or to specific relay(s).
5) Creates a comment to be used in a lightning payment that will go to the receiver. The comment will be encrypted before sending.
   - Comment content: [eventID] + [timestamp] + [senders_known_npub] + [onetime_npub] + [identity_token] + ["blastr" | [relay] | [relay_list]]
   - Creates signature of the comment (lightning_comment_signature) using onetime_nsec.
   - Encrypts the comment using receivers_known_npub (encrypted_lightning_comment).
     - Comment encryption could use alternative keys tied to the receiver (pgp, a bitcoin key, etc.).
6) Creates nostr DM event
   - kind: ?
   - author: onetime_npub
   - content: [lightning_comment_signature] + [end_sig_delimiter] + [encrypted_message]
   - Sets TTL
   - Signs event with onetime_nsec
7) Publishes nostr event
   - Regardless of method used (Blastr or specific relays), the npub will be new to all of the relays
     - Relay permissions: Sender needs write, Receiver needs read      
8) Sends lightning payment to message recipient
   - Minimum sat amount determined by ?
   - Comment construct: [DM_prefix] + [encrypted_lightning_comment]

## Receiving Scenario 1: (preferred) Lightning wallet is designed to work with a linked DM client
- Receiving wallet sees the DM prefix and passes the payment comment to the DM client.
- Proceed with DM Client Receiving (below).

## Receiving Scenario 2: The DM client is also a lightning wallet
- Proceed with DM Client Receiving (below).

## Receiving Scenario 3: Lightning wallet treats the payment like any other payment
- Receiver manually copies the lightning comment into their DM client.
  - Receiver will need to be aware of the lightning comment and know what to do with it.
- DM client provides a paste field.
- Receiver pastes encrypted comment to DM client.
- Proceed with DM Client Receiving (below).

## DM Client Receiving
- Client uses receivers nsec to decrypt the lightning payment comment.
- Client validates that the identity_token matches the senders_known_npub and onetime_npub values from the comment.
  - Validation confirms that the nsec matching the senders_known_npub was used to create the identity_token.
  - If the client has cached info for the senders_known_npub, then it may want to make a go-nogo decision here. If it doesn't have any cached info, then it SHOULD just move to the next step to get the message. The client SHOULD NOT make any external queries to get npub information at this point as that will leak information
- Client retrieves the event from a nostr relay.
- Client validates lightning_comment_signature in the unencrypted part of the event content against the comment received in the lightning payment.
  - Validation confirms that the lightning payment came from the event author.
- Client decrypts the encrypted part of the event content.
- Client specific activity follows but, as with any message, should probably include malware and spam protection on the content payload.
