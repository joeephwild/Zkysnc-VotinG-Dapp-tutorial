# zkSync Voting Application

This repository contains a blockchain-based voting application built using Solidity for smart contract development, zkSync Era for deployment, and React for the frontend interface.

## Prerequisites

Ensure you have the following tools installed:
- Node.js (v14 or later)
- npm (Node Package Manager)
- Solidity Compiler (solc)
- zkSync CLI
- Hardhat (Ethereum development environment)
- React (Frontend framework)

## Setting Up Your Development Environment

### 1. Initialize the Project

1. Create a new directory for your project:
    ```bash
    mkdir zkSyncVotingApp
    cd zkSyncVotingApp
    npm init -y
    ```

2. Install Hardhat and necessary plugins:
    ```bash
    npm install --save-dev hardhat @nomiclabs/hardhat-waffle @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy ethereum-waffle chai
    ```

3. Initialize a new Hardhat project:
    ```bash
    npx hardhat
    ```
    Choose "Create an empty hardhat.config.js" when prompted.

### 2. Write the Solidity Smart Contract

1. Create a `contracts` directory and add a file named `Voting.sol`:
    ```bash
    mkdir contracts
    touch contracts/Voting.sol
    ```

2. Open `Voting.sol` in your preferred text editor and add the following code:
    ```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.0;

    contract Voting {
        struct Candidate {
            string name;
            uint256 voteCount;
        }

        struct Voter {
            bool authorized;
            bool voted;
            uint256 vote;
        }

        address public owner;
        string public electionName;
        mapping(address => Voter) public voters;
        Candidate[] public candidates;
        uint256 public totalVotes;

        event ElectionResult(string candidateName, uint256 voteCount);

        modifier ownerOnly() {
            require(msg.sender == owner, "Owner only function");
            _;
        }

        constructor(string memory _name, string[] memory _candidates) {
            owner = msg.sender;
            electionName = _name;
            for (uint256 i = 0; i < _candidates.length; i++) {
                candidates.push(Candidate(_candidates[i], 0));
            }
        }

        function authorize(address _person) public ownerOnly {
            voters[_person].authorized = true;
        }

        function vote(uint256 _candidateIndex) public {
            require(!voters[msg.sender].voted, "You have already voted");
            require(voters[msg.sender].authorized, "You are not authorized to vote");

            voters[msg.sender].voted = true;
            voters[msg.sender].vote = _candidateIndex;

            candidates[_candidateIndex].voteCount += 1;
            totalVotes += 1;
        }

        function endElection() public ownerOnly {
            for (uint256 i = 0; i < candidates.length; i++) {
                emit ElectionResult(candidates[i].name, candidates[i].voteCount);
            }
        }

        function getCandidates() public view returns (Candidate[] memory) {
            return candidates;
        }
    }
    ```

### 3. Compile and Deploy the Smart Contract

1. Update `hardhat.config.js` to configure the Solidity compiler and zkSync settings:
    ```javascript
    require("@nomiclabs/hardhat-waffle");
    require("@matterlabs/hardhat-zksync-solc");
    require("@matterlabs/hardhat-zksync-deploy");

    module.exports = {
      zksolc: {
        version: "1.1.1",
        compilerSource: "binary",
        settings: {}
      },
      networks: {
        zkSyncTestnet: {
          url: "https://zksync2-testnet.zksync.dev",
          ethNetwork: "goerli", // The Ethereum testnet to use
          zksync: true
        }
      },
      solidity: "0.8.0",
      paths: {
        sources: "./contracts",
        tests: "./test",
        cache: "./cache",
        artifacts: "./artifacts"
      }
    };
    ```

2. Compile the smart contract:
    ```bash
    npx hardhat compile
    ```

3. Create a deployment script in the `scripts` directory. Name the file `deploy.js`:
    ```bash
    mkdir scripts
    touch scripts/deploy.js
    ```

4. Add the following code to `deploy.js` to handle the deployment process:
    ```javascript
    const { ethers } = require("hardhat");

    async function main() {
      const Voting = await ethers.getContractFactory("Voting");
      console.log("Deploying Voting contract...");

      const electionName = "Best Programming Language 2024";
      const candidates = ["Solidity", "Rust", "Python", "JavaScript"];

      const voting = await Voting.deploy(electionName, candidates);
      await voting.deployed();

      console.log("Voting contract deployed to:", voting.address);
    }

    main()
      .then(() => process.exit(0))
      .catch(error => {
        console.error(error);
        process.exit(1);
      });
    ```

5. Deploy the smart contract to zkSync Era testnet:
    ```bash
    npx hardhat run scripts/deploy.js --network zkSyncTestnet
    ```

### 4. Build the React Application

1. Initialize a new React application:
    ```bash
    npx create-react-app frontend
    cd frontend
    ```

2. Install ethers.js to interact with the Ethereum blockchain:
    ```bash
    npm install ethers
    ```

### 5. Connect the React Application to zkSync Era

1. Create a `.env` file in the `frontend` directory to store environment variables, including the deployed contract address and ABI (Application Binary Interface).

    `.env`:
    ```env
    REACT_APP_VOTING_CONTRACT_ADDRESS=your_deployed_contract_address
    REACT_APP_VOTING_CONTRACT_ABI=your_contract_abi
    ```

2. Create a `src/contracts` directory and save the ABI file as `Voting.json`.

3. In `src`, create a new file `Voting.js` to interact with the smart contract:
    ```javascript
    import React, { useState, useEffect } from 'react';
    import { ethers } from 'ethers';
    import VotingAbi from './contracts/Voting.json';

    const Voting = () => {
      const [candidates, setCandidates] = useState([]);
      const [electionName, setElectionName] = useState("");
      const [selectedCandidate, setSelectedCandidate] = useState(null);
      const [totalVotes, setTotalVotes] = useState(0);

      useEffect(() => {
        const fetchContractData = async () => {
          const provider = new ethers.providers.Web3Provider(window.ethereum);
          const signer = provider.getSigner();
          const contract = new ethers.Contract(
            process.env.REACT_APP_VOTING_CONTRACT_ADDRESS,
            VotingAbi.abi,
            signer
          );

          const name = await contract.electionName();
          setElectionName(name);

          const candidatesArray = await contract.getCandidates();
          setCandidates(candidatesArray);

          const votes = await contract.totalVotes();
          setTotalVotes(votes.toNumber());
        };

        fetchContractData();
      }, []);

      const handleVote = async () => {
        if (selectedCandidate !== null) {
          const provider = new ethers.providers.Web3Provider(window.ethereum);
          const signer = provider.getSigner();
          const contract = new ethers.Contract(
            process.env.REACT_APP_VOTING_CONTRACT_ADDRESS,
            VotingAbi.abi,
            signer
          );

          await contract.vote(selectedCandidate);
          const votes = await contract.totalVotes();
          setTotalVotes(votes.toNumber());
        }
      };

      return (
        <div>
          <h1>{electionName}</h1>
          <ul>
            {candidates.map((candidate, index) => (
              <li key={index}>
                {candidate.name} - {candidate.voteCount.toNumber()} votes
                <button onClick={() => setSelectedCandidate(index)}>Vote</button>
              </li>
            ))}
          </ul>
          <button onClick={handleVote}>Submit Vote</button>
          <p>Total Votes: {totalVotes}</p>
        </div>
      );
    };

    export default Voting;
    ```

4. Update `src/App.js` to include the `Voting` component:
    ```javascript
    import React from 'react';
    import Voting from './Voting';

    function App() {
      return (
        <div className="App">
          <Voting />
        </div>
      );
    }

    export default App;
    ```

## Conclusion

You have successfully built and deployed a blockchain-based voting application using Solidity and zkSync Era, and created a React frontend to interact with the smart contract. From here, you can further enhance the application by adding more features such as user authentication, vote results visualization, and more robust UI/UX design. Happy coding!
