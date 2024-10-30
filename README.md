'use strict';

const { Contract } = require('fabric-contract-api');

class AssetContract extends Contract {

    // 1. Create an Asset (Account)
    async CreateAsset(ctx, dealerId, msisdn, mpin, balance, status) {
        const asset = {
            DEALERID: dealerId,
            MSISDN: msisdn,
            MPIN: mpin,
            BALANCE: parseFloat(balance),
            STATUS: status
        };

        // Check if the asset already exists
        const exists = await this.AssetExists(ctx, dealerId);
        if (exists) {
            throw new Error(`The asset ${dealerId} already exists`);
        }

        // Store asset on the ledger (world state)
        await ctx.stub.putState(dealerId, Buffer.from(JSON.stringify(asset)));
    }

    // 2. Read the Asset Details
    async ReadAsset(ctx, dealerId) {
        const assetJSON = await ctx.stub.getState(dealerId); 
        if (!assetJSON || assetJSON.length === 0) {
            throw new Error(`The asset ${dealerId} does not exist`);
        }
        return assetJSON.toString();
    }

    // 3. Update the Asset (Account) Details
    async UpdateAsset(ctx, dealerId, newBalance, newStatus, transAmount, transType, remarks) {
        const assetJSON = await this.ReadAsset(ctx, dealerId);
        const asset = JSON.parse(assetJSON);

        // Update asset fields
        asset.BALANCE = parseFloat(newBalance);
        asset.STATUS = newStatus;

        // Record the transaction details
        const transaction = {
            TRANSAMOUNT: parseFloat(transAmount),
            TRANSTYPE: transType,
            REMARKS: remarks,
            TIMESTAMP: new Date().toISOString()
        };

        // Store updated asset on the ledger
        await ctx.stub.putState(dealerId, Buffer.from(JSON.stringify(asset)));

        // Add transaction history to the blockchain ledger (immutable record)
        const transactionKey = ctx.stub.createCompositeKey('Transaction', [dealerId, ctx.stub.getTxID()]);
        await ctx.stub.putState(transactionKey, Buffer.from(JSON.stringify(transaction)));
    }

    // 4. Check if an Asset Exists
    async AssetExists(ctx, dealerId) {
        const assetJSON = await ctx.stub.getState(dealerId);
        return assetJSON && assetJSON.length > 0;
    }

    // 5. Get the Transaction History for a specific asset
    async GetTransactionHistory(ctx, dealerId) {
        const resultsIterator = await ctx.stub.getHistoryForKey(dealerId);
        const transactions = [];

        while (true) {
            const res = await resultsIterator.next();

            if (res.value) {
                const transaction = {
                    TxId: res.value.txId,
                    Timestamp: res.value.timestamp,
                    Value: JSON.parse(res.value.value.toString('utf8'))
                };
                transactions.push(transaction);
            }

            if (res.done) {
                await resultsIterator.close();
                return JSON.stringify(transactions);
            }
        }
    }
}

module.exports = AssetContract;
