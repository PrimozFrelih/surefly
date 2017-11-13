# SureFly

This repo is the codebase for the crowdsourced decentralized insurance platform SureFly developed for proffer hackathon

There are two parts of this project

* A Toshi Bot project: UI for the platform using a Toshi Bot
* A Truffle project for the smart contracts


Using this app you can

* Insure your flight from misses
* Invest in people's insurance policy

Read our [problem statement and solution](https://docs.google.com/document/d/1M0w_6ArvHGOpiro6yM1p4IM7u9SvdDZ-HENzLx1EttY/edit?usp=sharing).

## How to use

* Download the Toshi App
* Search for SureFlyBot
* Click on message: You are good to go! 


## Basic Process Flow:

For a person who wants to insure his/her flight (Will be called Seeker in all future references):
 * Seeker enters personal and flight details 
 * Identity is confirmed using Aadhar and flight ticket data is recieved from the airlines
 * Based on the time of booking the insurance and his location, a probability is generated by the system, of him missing the flight
 * A maximum premium amount is calculated on the basis of this probability
 * The seeker is asked the maximum and minimum amount that he wants to insure and is asked to pay the max amount he enters as the premimum

For a person who wants to invest in other people's investment policies (Will be referred to as Investor):
  * The investor is shown a list of policies open for investment, along with all the details like the calculated probability, ETA to airport, present pool size for each policy.
  * He chooses the one he wants to invest in and is then asked for the amount which he is asked to pay. The ratio of this amount to the maximum payout, decides his share of returns of the premium.
  * Thus, with a number of investors, the pool starts getting filled. In case, the minimum amount is not met, the policy stands cancelled and everyone gets their money back.

A more detailed processflow can be found [here](https://realtimeboard.com/app/board/o9J_k0XST14=/)

## SureFlyBot : The chatbot on Toshi as the front end

Here are a few screenshots from the conversation of a seeker with the bot:

<a href="https://postimages.org/" target="_blank"><img src="https://s18.postimg.org/tvm3efwvd/ss00.png" alt="ss00"/></a> <a href="https://postimages.org/" target="_blank"><img src="https://s18.postimg.org/52cjdtvvd/ss01.png" alt="ss01"/></a> <a href="https://postimages.org/" target="_blank"><img src="https://s18.postimg.org/aqiu4z809/ss03.png" alt="ss03"/></a><br/><br/>

This bot makes use of three APIs:
  * **The Aadhar API**: To verify a person's name against his aadhar number and a mobile OTP verification for identity verification
  * **A flight data API**: To take the booking number and name as an input and
    * Ticket identity verification: If the ticket is indeed booked in his name
    * Provide details like ticket price, flight departure time, boarding gate closure time
    * The API is hit at the estimated boarding gate closure time, which returns, if the passenger has boarded the flight (using the confirmation at the boarding gate). Also, if the flight is delayed, the API is hit at the delayed flight boarding gate closure time and again the same process is repeated. 
    * On every API hit, two boolean values, if the flight is cancelled and if the ticket is cancelled are also returned.
  * **Google Maps API**: The google maps API returns estimated time of arrival at the airport, taking the time, location (geodata) and airport name as inputs

The functionalities of this bot, other than taking user inputs and calling these APIs are:
  * Calculating probability based on ETA provided by Google maps and the time of booking.
  * Calculating the premium based on this probability and the max payout amount.
  * It makes the following web3 calls to the smart contract:
    * Adding User: calls addUser() using all the user data accumulated to add a new user
    * Calls various functions triggering payouts, like flight departure, boarding gate closure, flight cancellation, ticket cancellation. There are separate functions for each of these in the smart contract and they are called using web3.js, triggering payouts.
    * Post-Payout, recieving all all the data from the smart contract for presenting back to the users, like payout amounts, net payments etc.
    * Stores all this data on a MongoDB, for generating a data source which will help improve the probability function in the future.

## SureFly.sol: The deployed Smart Contract

The smart contract is presently deployed on the testnet Ropsten, [here.](https://ropsten.etherscan.io/address/0x013b753cad4193c19f50c507cefd8aee65ece051)

This smart contract is responsible for doing all the transactions, based on different conditions being met. The claim process is automated using this smart contract. It is deployed on the testnet and we use a Infura address to communicate with it using Web3. 

Major State Variables:
  * A structure to store all the details of a seeker. Mappings of this structure are used to map different structure instances to an id number and a address.
    * This structure has a mapping of investor addresses with their respective amounts and id, as a variable.
  * An Enum that has the different states a policy can be in. An instance of this enum is a variable in the seeker structure.
  * Count variables to keep a count and to iterate through user structure mapping and the investor mapping. 

A few major functions of this contract:
  * **addUser(uint256 _maxPayout, uint _minPayout, uint256 _initialPremium, uint256 _prob)** : Used to add a new policy (seeker). Default policyStatus is OPEN
  * **isMinRaised(uint _id)** : Internally used to check if min amount to be raised is reached for a particular policy
  * **queryPoolSize(address adr)** : Externally called to get the current pool size of a particular seeker.
  * **listAllAvailablePolicies()** : Externally called to get the list of all open policies. Returns an array of addresses.
  * **invest(address _adr, uint256 _amount)** : Called whenever a new investor is added. Certain conditions are checked before the investment request is forwarded, like the flight shouldn't have departed, Max Payout shouldn't have been reached, the user must not have boarded etc. Adds address and amount invested to the storage for the particular policy
  * **payout(uint id)** : Called whenever there is a trigger condition for the payout. These are the four conditions and what happens in each case:
    * Flight Missed: The seeker is paid the amount that has been raised in the pool. The investors are paid the premium which is divided proportionally on the basis of their contribution to the pool.
    * Flight Not Missed: The seeker is paid nothing. The investors get their investment back along with the premium divided in proportionally on the basis of their contribution to the pool.
    * Flight Cancelled/Minimum Insured amount not met: The seeker gets back his premium. The investors get their respective investments back.
    * Ticket Cancelled : The seeker loses his premium and doesn't get a payout. Investors get their invested amount back along with the premium divided proportionally. 

    All this functionality is implemented in this function. For ether transfers, a separate function trasferEther() is called for each of the cases.

## Probability Calculation

The probability calculation based on time and location data is a study in statistics on the following:
  * Historic miss rates, Close miss rates
  * Historic flight delay rates
  * Individual miss rates
  * Correlation between expected time of arrival and probability of missing
  * Classification by flight, route, time, city

For a precise probability calculation, one needs a extensive data pool of the following data:
  * Route-wise correlation between "Minutes early/late" and Flight misses
  * Parameters could include - city, flight, whether flight got delayed, by how much time

After some research, due to the lack of data pools on flight metrics, we generated the following sample data, which we believe is close to real world scenarios:

[![Screen_Shot_2017-11-13_at_11.07.17.png](https://s8.postimg.org/7v5slc3h1/Screen_Shot_2017-11-13_at_11.07.17.png)](https://postimg.org/image/e8uvol8cx/)

We've used this in calculating the probability.

**Sources:**

  * [An Analysis of Passenger Delays Using Flight Operations and Passenger Booking Data](http://isapapers.pitt.edu/56/1/2004-20_Bratu.pdf)
  * [A model for estimating airline passenger trip reliability metrics from system-wide flight simulations] (http://www.scielo.br/scielo.phpscript=sci_arttext&pid=S2238-10312013000200017)
  * [Flight and Passenger Delays](http://web.mit.edu/airlines/industry_outreach/board_meeting_presentation_files/meeting-nov-2008/Barnhart%20Flight%20and%20Passenger%20Delays.pdf)

## Future Prospects

This is the simplest use case of a automated crowdfunded insurance program.

This itself can be extended to provide automatic insurance coverage for all the paseensgers in a particular flight, each being a seeker and an investor at the same time. This will reduce overall premium and the avg probability across all passengers will be approx. 2%.

Such a system can be implemented in any kind of parametric insurance like crop insurances. This can also be extended to use cases like suppply chain goods insurance, where all the stakeholders insure the travelling good, once a stable supply chain tracking system in place. 