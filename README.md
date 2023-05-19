# StoreKit2Demo
基于苹果官方的storekit2demo练习

### 前置条件

- 需要自己修改bundle ID
- 需要在开发者账号中创建内购
- Products中填入自己的内购ID

### 内容

根据内购ID获取内购的信息（可用来做第一次拦截）

```
let storeProducts = try await Product.products(for: productIdToEmoji.keys)
            
            print(storeProducts)
            var newCars: [Product] = []
            var newSubscriptions: [Product] = []
            var newNonRenewables: [Product] = []
            var newFuel: [Product] = []
            
            //Filter the products into categories based on their type.
            for product in storeProducts {
                switch product.type {
                case .consumable: // 消耗
                    newFuel.append(product)
                case .nonConsumable:// 非消耗
                    newCars.append(product)
                case .autoRenewable: // 自动订阅
                    newSubscriptions.append(product)
                case .nonRenewable: // 非续期订阅
                    newNonRenewables.append(product)
                default:
                    //Ignore this product.
                    print("Unknown product")
                }
            }
```

购买

```
    func purchase(_ product: Product) async throws -> Transaction? {
        //Begin purchasing the `Product` the user selects.
        print("iap: start pay")
        let result = try await product.purchase()
        print("iap: \(result)")
        switch result {
        case .success(let verification):
            let transaction = try await verifiedAndFinish(verification)
            return transaction
        case .userCancelled: // 用户取消
            print("iap: user cancel")
            return nil
        case .pending: // 此次购买被挂起
            print("iap: pending")
            return nil
        default:
            return nil
        }
    }
```

校验

```
    func verifiedAndFinish(_ verification:VerificationResult<Transaction>) async throws -> Transaction?{
        //Check whether the transaction is verified. If it isn't,
        //this function rethrows the verification error.
        let transaction = try checkVerified(verification)
        
        //The transaction is verified. Deliver content to the user.
        // 这里将订单提交给服务器进行验证 ~~~
        await updateCustomerProductStatus()
        
        //Always finish a transaction.
        await transaction.finish()
        print("iap: finish")
        return transaction
    }
```

监听(可放在主函数中，用来判断以前未支付成功的订单)

```
    func listenForTransactions() -> Task<Void, Error> {
        return Task.detached {
            //Iterate through any transactions that don't come from a direct call to `purchase()`.
            // 修改update 为 unfinished
            for await result in Transaction.updates { //会导致二次校验？
                do {
                    
                    print("iap: updates")
                    print("result:\(result)")
                    let transaction = try await self.verifiedAndFinish(result)
                } catch {
                    //StoreKit has a transaction that fails verification. Don't deliver content to the user.
                    print("Transaction failed verification")
                }
            }
        }
    }
```

