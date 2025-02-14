# Notes
- player can enter the raffle by calling `enterRaffle` function, multiple addresses can enter the raffle but duplicate addresses can't enter the raffle
- player can get refund on ticket and value if they call `refund` function
- every few seconds raffle draws a winner and mints a random NFT
- the owner of the raffle will set the feeAddress to take a cut of value and rest is sent to the winner of the raffle


- `selectWinner` picks the winner, there is a raffle duration

# High

- DoS


# Attack Vectors