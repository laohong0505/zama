# zama
setup_project.sh
#!/bin/bash
set -e
  
PROJECT="zama-fhe-dapp"

echo "📦 正在创建项目目录 $PROJECT ..."
rm -rf $PROJECT
mkdir -p $PROJECT/{contracts,scripts,frontend/src,frontend/abis}

########################################
# 合约文件
########################################
cat > $PROJECT/contracts/EncryptedStorage.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@fhenixprotocol/contracts/FHE.sol";

contract EncryptedStorage {
    euint64 private storedNumber;

    event NumberStored(euint64 encryptedValue);
    event NumberAdded(euint64 encryptedResult);

    function store(euint64 encryptedValue) public {
        storedNumber = encryptedValue;
        emit NumberStored(encryptedValue);
    }

    function add(euint64 encryptedValue) public {
        storedNumber = FHE.add(storedNumber, encryptedValue);
        emit NumberAdded(storedNumber);
    }

    function getEncrypted() public view returns (euint64) {
        return storedNumber;
    }
}
EOF

########################################
# Hardhat 配置
########################################
cat > $PROJECT/hardhat.config.js << 'EOF'
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20",
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};
EOF

########################################
# 部署脚本
########################################
cat > $PROJECT/scripts/deploy.js << 'EOF'
const hre = require("hardhat");

async function main() {
  const EncryptedStorage = await hre.ethers.getContractFactory("EncryptedStorage");
  const contract = await EncryptedStorage.deploy();
  await contract.deployed();
  console.log("EncryptedStorage deployed to:", contract.address);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
EOF

########################################
# 交互脚本
########################################
cat > $PROJECT/scripts/interact.js << 'EOF'
const hre = require("hardhat");

async function main() {
  const contractAddr = process.env.CONTRACT_ADDR;
  const EncryptedStorage = await hre.ethers.getContractAt("EncryptedStorage", contractAddr);

  console.log("Contract at:", contractAddr);

  const value = await EncryptedStorage.getEncrypted();
  console.log("Encrypted value:", value);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
EOF

########################################
# 前端 package.json
########################################
cat > $PROJECT/frontend/package.json << 'EOF'
{
  "name": "zama-fhe-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "ethers": "^6.7.0"
  },
  "scripts": {
    "start": "react-scripts start"
  }
}
EOF

########################################
# 前端 App.js
########################################
cat > $PROJECT/frontend/src/App.js << 'EOF'
import React, { useState } from "react";
import { ethers } from "ethers";
import EncryptedStorageABI from "../abis/EncryptedStorage.json";

const CONTRACT_ADDR = process.env.REACT_APP_CONTRACT_ADDR;

function App() {
  const [status, setStatus] = useState("");

  async function connectWallet() {
    if (!window.ethereum) return alert("No wallet detected!");
    await window.ethereum.request({ method: "eth_requestAccounts" });
    setStatus("Wallet connected.");
  }

  async function getEncrypted() {
    const provider = new ethers.BrowserProvider(window.ethereum);
    const contract = new ethers.Contract(CONTRACT_ADDR, EncryptedStorageABI, provider);
    const value = await contract.getEncrypted();
    setStatus("Encrypted value: " + value.toString());
  }

  return (
    <div style={{ padding: "2rem" }}>
      <h2>Zama FHE DApp</h2>
      <button onClick={connectWallet}>Connect Wallet</button>
      <button onClick={getEncrypted}>Get Encrypted</button>
      <p>{status}</p>
    </div>
  );
}

export default App;
EOF

########################################
# 前端 index.js
########################################
cat > $PROJECT/frontend/src/index.js << 'EOF'
import React from "react";
import ReactDOM from "react-dom";
import App from "./App";

ReactDOM.render(<App />, document.getElementById("root"));
EOF

########################################
# README
########################################
cat > $PROJECT/README.md << 'EOF'
# Zama FHE DApp

本项目基于 [Zama FHEVM](https://docs.zama.ai/) 教程，包含：
- FHE 智能合约
- Hardhat 部署脚本
- React 前端交互

## 使用步骤

1. 安装依赖：
```bash
./setup.sh
