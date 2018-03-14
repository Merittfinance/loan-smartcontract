# Loan Smartcontract

_a smart contract for loans_


**Note:** technically this smart contract should probably be referred to as _payment-conduit_ rather than as loan because (a) technically that is its purpose, and (b) the range of applications is much wider than loans; for example, this smart contract could be used for equity-like contracts, or even derivatives. In the following however we will assume for simplicity purposes that the contract is a loan.

## Description

### Creating a loan contract

A loan contract is created by a *borrower* (who in a capital markets-like context might also be referred to as *issuer*) who then sells the tokens to the *lenders* (who in a capital markets-like context might also be referred to as *investors*). The mechanics can look at bit complicated at first, but is actually pretty simple: in the traditonal world of finance it is akin to first creating the record of a future liability, eg by writing a cheque that can be cashed at some date in the future, and then selling the cheque to someone else, probably at a discount to account for the interest payment. So if the example below is hard to follow it is worth to think of the tokens as cheques.

So when a borrower creates a loan contract, he or she has to first fix the following parameters

- amount

- interest rate

- repayment profile (installments, interest-only, zero-coupon)

- maturity

Now ideally the interest rate of the loan would be set at the correct interest rate for the borrower in question. This poses a problem as the correct interest rate is only known when a lender is found, ie after the loan is created. There are two solutions to this particular problem: the first one is that the borrower and the lender(s) first agree in principle on the fair rate, and the borrower bases the interest rate at which he or she creates the loan on this. The second solution is that the borrower creates a loan at an estimated rate, and the effective rate will be determined by the price at which the tokens are eventually sold.

Once the borrower has chosen all the above parameters, the contract is deployed on the blockchain. This records the liability of the borrower, and at the same time creates the tokens that represent the assets balancing this liability, the *cheques* in the above traditional-world example, and records (the address of) the borrower as the owner of all tokens. Those tokens are referred to as *retained tokens* and are described in more detail in the section below. To cut a long story short however, retained tokens do represent a situation where the borrower makes a payment to him or herself, so they can be netted out when looking at the liability under this contract.

In principle, retained tokens do not require a special treatment: the borrower could simply pay Ether into the contract, and the proportion corresponding to the retained tokens would immediately sent back. This however is sub-optimal. Consider for example a 100ETH loan contract with 100 tokens where only 10 tokens are distributed to third parties, and 90 tokens are retained. Even though the borrower only owes 10ETH to external parties, he or she nevertheless must make a payment of 100ETH, just to have 90ETH returned immediately. At best this is inconvenient if the borrower does not have that amount of Ether on the relevant address, and at worst he or she can't make the payment because of lack of funds.


### Distributing a loan contract

Loan tokens are standard ERC20 tokens, so they can be distributed using any of the mechanisms that allow for the distribution and trading of ERC20 tokens. In particular, the Meritt wallet will implement a mechanism that allows to sell those tokens to another wallet in an atomic manner, ie either both the token and the agreed Ether amount are exchanged, or no transaction at all takes place.


### Terminating a loan contract

A loan contract will not usually terminate in the sense that the tokens will not be destroyed--there will simply be a mechanism to understand when a contract can be considered repaid, in which case the tokens will become worthless and can be removed from view. In principle, terminated contracts could be killed, but this requires an unequivocal decision that a contract has indeed been repaid (which might not always be easy to do, eg in case of payment delays where it is not clear whether additional interest accrued). In an environment where defaults are possible it is very hard to programmatically decided whether a contract has been repaid in all possible case, and allow this decision to be taken via an external agent / oracle creates a security risk, so it is probably best to keep the contract alive forever.

There is on exception to this rule however, and this is when all tokens are owned by the borrower, just after the contract has been created. In this case the contract can be killed without consequences, and it is advantageous to permit this, to allow for the deletion of contracts that for example have been created with erroneous parameters, or where no lender can be found.


### Retained tokens

As already mentioned above, *retained tokens* are tokens that are owner by the (registered address of) the borrower, so they represent a situation where the borrower has to pay him or herself. When looking at the (net) liability under this contract those retained tokens can be ignored, and therefore initially when _all_ tokens are owned by the borrower the net liability is zero, as it should be because at this point no lending contract has been established.


## Staking mechanism

Every loan contract requires the staking of a MTT tokens, both from the borrower and the lender. The exact mechanism is yet to be determined, but the idea is that upon creation of the contract, both the borrowers and the lenders stake must be transferred into the contract, the stake being proportional to the payments required under the contract plus a fixed component for the borrowers. Whenever a payment is made into the contract, a commensurate proportion of the staking tokens is returned to both the borrower and the lenders. The lender fixed component is returned upon surrendering their tokens. When [90%] of lender tokens have been surrendered, all staking tokens are returned and the contract is killed.



## Interfaces

### ERC20 Interface

The contract implements an interface that is compliant with the ERC20 token standard (described [here][ERC20]). That means it implements the following functions

- `totalSupply`
- `balanceOf(address _owner) constant returns (uint256 balance)`
- `transfer(address _to, uint256 _value) returns (bool success)`
- `transferFrom(address _from, address _to, uint256 _value) returns (bool success)`
- `approve(address _spender, uint256 _value) returns (bool success)`
- `allowance(address *_owner*, address *_spender*) constant returns (uint256 remaining)`

and the following events

- `Transfer(address indexed _from, address indexed _to, uint256 _value)`
- `Approval(address indexed _owner, address indexed _spender, uint256 _value)`


### Payment Conduit Interface

The payment conduit interface is the interface that implements the specific functionality that is needed to model loans and other financial instruments.

Most importantly, _payments into the contract are forwarded to token holders on a pro-rata basis_. There are a number of issues to consider in this respect

- only payments coming from _approved addresses_ (which, in the case of loans, would be the address of the borrower) should be forwarded to the token holders; all other payments are returned to the sender

- if payments can not be evenly split into payments for the respective tokenholders then the maximum possible amount is forwarded, and the remainder is either returned to the sender, or retained in the contract (tbd)

- if tokens are held by one of the approved addresses then those tokens are excluded from the distribution of funds; from a bookkeeping point of view however this is recorded as a payment to self (see _retained tokens_ section above)

**Note on fund forwarding**. Ideally this forwarding is automatic. Current Ethereum gaz limits however might not make it possible to forward payments to a large number of recipients because in this case hard gaz limits could be hit. An alternative mechanism is therefore to keep all Ether that has been submitted from the right address in the contract, and have a function that token holders can call to withdraw.

In addition, the following functions are available

**payments**. The `payments` function returns a list of all payments that have been effectuated through the contract. The amount is grossed up to account for retained tokens, ie the amount is shown as if the entire gross payment had been made into the contract, and the retained tokens had received their payment. The returned object is as follows

    [
        {height: <block height>, amount: <payment amount>},
        {height: <block height>, amount: <payment amount>},
        ...
        {height: <block height>, amount: <payment amount>},
    ]

**recipients**. The `recipients(height)` function returns a list of addresses that have received a payment at blockheight _height_ together with the number of tokens the address held for that particular payment. The list contains retained tokens, and it returns its data as follows

    [
        {addr: <recipient address>, tokens: <number of tokens>},
        {addr: <recipient address>, tokens: <number of tokens>},
        ...
        {addr: <recipient address>, tokens: <number of tokens>},
    ]

**details**. The `details` function returns (static) details of the underlying contract. Details are yet to be determined, but the result of the function call might look as follows:

    {
        /* generic data fields */
        version: "1.0"                      /* the version of the smart contract */

        payer: <payer address>              /* the Ethereum address of the borrower */

        type_f: "loan"                      /* friendly name of the contract type */
        subtype_f: "installment loan"       /* friendly name of the contract subtype */
        type_ns: "types.meritt.co"          /* namespace URL for the contract definition */
        type_nm: "inst"                     /* exact type of the contract (unique in namespace) */

        /* this URL is pointing to exact contract terms of this specific contract */
        contract: "https://types.meritt.co/inst?addr=<contract addr>"                    

        /* loan-specific data fields */
        principal: <principal amount>       /* the principal amount outstanding at origination */
        startdt: <start date>               /* start date of the loan */
        enddt: <end date>                   /* end date of the loan */
        freq: <frequency>                   /* payment frequency (d,w,m,q,h,a) */
        schedule: [
            /* note: amount is the total amount, princ is the principal repayment included */
            {date: <payment datetime>, amount: <payment amount>, princ: <principal repayment>},
            {date: <payment datetime>, amount: <payment amount>, princ: <principal repayment>},
            ...
            {date: <payment datetime>, amount: <payment amount>, princ: <principal repayment>},
        ]
    }

**killNewContract**. The `killNewContract` function allows to kill a freshly created contract. It only works if firstly all tokens are owned by the payer, ie none have been distributed (or all have been bought back) and secondly no payments have been registered in the contract (see `payments`). The purpose of this function is solely to clean up unused and/or erroneously created loan contracts. It is not meant for killing contracts after repayment.

**surrenderTokens**. The `surrenderTokens` function cancels all tokens held by the caller. It will usually only be called when the contract has been terminated, and it triggers the transfer of the MTT staking tokens associated to those loan tokens to the lender.


**withDraw**. In case it is not possible to forward funds automatically to all tokenholders, the contract provides a `withdraw` function that triggers the contract to send all Ether currently held in the contract that belongs to the tokens currently held by the caller. Note that this calculation is rather complex: imagine for example the case where A and B hold 100 and 150 tokens respectively when a payment occurs. Person A calls the `withdraw` function and then transfers all of his the tokens to B. After that, B calls the `withdraw` function. Despite at this stage holding 250 tokens, the algorithm must ensure to only pay out on the 150 tokens B initially held.




### Voting Interface

The contract should also introduce a voting interface, which allows token holders to vote on proposal, eg with respect to a restructuring in case of default. It remains to be seen to which extent the interaction of the voting contract with the loan contract can be automated, for the time being it remains completely disjoint.

**castVote**. The `castVote(proposal_url, proposal_hash, choice)` casts the vote 'choice' with respect to the proposal that is located at the URL `proposal_url` (and whose hash is `proposal_hash` to ensure that the proposal can not be modified during the vote)

[ERC20]: https://theethereum.wiki/w/index.php/ERC20_Token_Standard
[ERC20wp]: https://en.wikipedia.org/wiki/ERC20
