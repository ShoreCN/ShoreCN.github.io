---
title: 区块链地址标准及编码解码
date: 2018-05-04 23:29:53
categories:
- 区块链
tags:
- 加密货币
- TRON
- 以太坊
---

区块链地址标准及编码解码
## 数据类型-地址 address
在solidity里的address等价于 uint160, 即长度20字节的数值  
在大部分开发语言内部通常使用十六进制格式进行操作(fe1f0e6efead80045af5ddb58b664fb6492230e3)  
有时显示时加上0x标明这是一个十六进制格式(0xfe1f0e6efead80045af5ddb58b664fb6492230e3)  

## CheckSum格式字符串
为了防止输入错误或者其他复制、粘贴、传输地址等情况出现失误, EIP-55制定了一套地址编码规则  
样例: 0xfE1f0e6EFEad80045Af5ddb58b664fb6492230E3  

### 产生规则
1. 将地址转换为十六进制结果(fe1f0e6efead80045af5ddb58b664fb6492230e3)
进行一次Keccak256哈希处理, 得到参照表(1ea1041c8817750ddd7e542a4754354676f5deb7ce20334eccf0936e81ba8726)
2. 遍历十六进制结果的每个字符,数字跳过, 字母处理, 使用参照表里相同位置的数值进行判断, 如果小于8则不变化, 大于8则变成大写字母(fE1f0e6EFEad80045Af5ddb58b664fb6492230E3)
3. 加上0x头, 得到最终结果(0xfE1f0e6EFEad80045Af5ddb58b664fb6492230E3)
## ETH地址
使用如上规则
## TRON地址
### 支持Hex格式地址  
(官方文档说法) 同一个私钥生成的Hex格式TRON地址和以太坊地址相比，除了最前面加上41，其他完全一样  
**所以TRON地址是21字节长度**

### 支持Base58Check格式地址
(官方文档说法) 对Hex格式的地址进行basecheck计算得到Base58格式地址，所有账户地址的第一个字符均为T  
样例: TWEkjtFFeVtfsdRT32Xx5KuWGcskYfhWjX  
> 地址前面添加版本前缀可以影响编码结果的前导符号

### 产生规则
1. 取版本字段(41)和有效负载信息(de533fbbb750d302728d3ad29d31e070bc14af4b)，并将它们连接在一起(41de533fbbb750d302728d3ad29d31e070bc14af4b)
2. 对第一步结果进行两次SHA256变换，取结果的前4个字节, 作为校验内容(6421c276)
3. 将第一步结果和第二步结果连接在一起(41de533fbbb750d302728d3ad29d31e070bc14af4b6421c276)
4. 进行base58 encode, 得到结果(TWEkjtFFeVtfsdRT32Xx5KuWGcskYfhWjX)

**但是在 TVM 虚拟机和 Solidity 编程中，需要再去掉前缀 41 成为和以太坊兼容的20字节地址格式**

## Application Binary Interface(ABI)
ABI即应用二进制接口, 作为与智能合约交互的标准, 起到交互数据的作用

### 结构
数据由函数选择器和参数编码组成
- 函数选择器Function Selector  
函数签名的 Keccak（SHA-3）哈希的前 4 字节  
基础原型即是函数名称加上由括号括起来的参数类型列表，参数类型间由一个逗号分隔开，且没有空格  
`例如transfer(address,uint256)`
- 参数编码  
从第5字节开始是被编码的参数  
动态类型(bytes,string,list,tuple)和静态类型(其他)  
编码规则  
    - 转换为十六进制之后补齐到32字节长度

### 第三方库
后端Python使用eth-abi库进行编码解码
iOS
Android

### 调用链条
#### ETH
- 前端-SDK-编码-发送请求至节点服务商-区块链
- 后端-Web3 SDK-编码-发送请求至节点服务商-区块链

#### TRON
- 前端-SDK-编码-发送请求至节点服务商-区块链
- 后端-编码(eth-abi)-发送请求至节点服务商-区块链
    - 编码过程中将TRON地址的首字节去除
    - logic.contract_tron_logic.TronContractInterface.execute_contract_func -> trx_abi.encode_single

## TRON 地址的特殊处理
由于 EVM 中 address 为 20 字节 bytes160 类型，而 TRON 地址的 hex 地址为 21 字节， 所以在传入参数的时候需要截取后 20 字节

**TVM 对 21 字节地址只有有限的支持，建议不要依赖此特性**

## 衍生
bytes数据格式, 可转换为十六进制的字符串进行传递, 不需要加0x头

## 参考产品
- TronLink  
进行Token转账时产生的data数据里面地址没有41前缀
- imToken  
未测试

## 参考资料

> ERC-55: Mixed-case checksum address encoding
> https://eips.ethereum.org/EIPS/eip-55

> 应用二进制接口Application Binary Interface(ABI) 说明
> https://solidity-cn.readthedocs.io/zh/develop/abi-spec.html

> TRON协议-账户地址格式
> https://cn.developers.tron.network/docs/> account#%E8%B4%A6%E6%88%B7%E5%9C%B0%E5%9D%80%E6%A0%BC%E5%BC%8F

> TRON参数编码和解码
> https://cn.developers.tron.network/docs/parameter-encoding-and-decoding

> Solidity ABI - TRON 地址的特殊处理TRON 地址的特殊处理
> https://andelf.gitbook.io/tron/smart-contract/abi

> List of address prefixes
> https://en.bitcoin.it/wiki/List_of_address_prefixes