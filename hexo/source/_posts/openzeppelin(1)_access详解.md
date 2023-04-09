---
title: openzeppelin(1)_assess详解
tags:
  - 区块链
  - openzeppelin
categories:  区块链
date: 2020-08-30 16:04:21
---

## Ownable.sol

合同模块提供了一个基本的访问控制机制，其中有一个账户（所有者），可以被授予对特定功能的独家访问。

`Ownable.sol`合约主要是定义了一个合约的拥有者字段`_owner`,在使用上

- 通过`_transferOwnership`方法来设置合约拥护者。
- 通过函数构造器`onlyOwner`来限制方法是否是合约拥有者调用。

```typescript
abstract contract Ownable is Context {
    address private _owner;
    // 事件
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        _transferOwnership(_msgSender());
    }
    
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    function owner() public view virtual returns (address) {
        return _owner;
    }

    
    function _checkOwner() internal view virtual {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
    }

   
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }

   
    function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```

## AccessControl.sol

`AccessControl`合约实现基于角色的访问控制机制。当我们需要在合约中定义不同地址拥有不同权限时会使用这个合约。

合约中主要通过`_roles`字段来记录某个角色下的用户列表，并且某个角色下面又定义了他对应的ADMIN的权限。如图所示

![](openzeppelin(1)_access详解/1.png)

在使用上，假设我们现在有一个ERC721合约，对于合约里面的方法`mint`,`burn`我们希望用不同的角色来运营，所以我们定义了`MINT_ROLE`和`BURN_ROLE`两个角色，并且两个角色里有对应的账户用来操作方法，所以address001可以使用mint方法，但没有burn方法的权限；address002可以使用burn方法，但没有mint方法的权限。

- mint -> MINT_ROLE [address001,address003]
- burn -> BURN_ROLE [address002,address004]

那谁有权限来对`MINT_ROLE`和`BURN_ROLE`角色进行添加账户和删除账户了，所以我们需要再定义一个角色，也就是上面的`adminRole`，我们定义了`ADMIN_ROLE`角色，并且将address000加入到角色中，同时我们使用`_setRoleAdmin('MINT_ROLE','ADMIN_ROLE') _setRoleAdmin('BURN_ROLE','ADMIN_ROLE')`将两个角色的adminRole设置成ADMIN_ROLE角色，这样的话address000就有了添加账户和删除账户的权限

- ADMIN_ROLE [address000]

主要方法如下 :

- getRoleAdmin : 获取角色的ADMIN角色
- grantRole : 设置角色， 具体看代码
- onlyRole : 函数构造器，用来判断是否有某个角色权限 

```typescript
abstract contract AccessControl is Context, IAccessControl, ERC165 {
    struct RoleData {
        mapping(address => bool) members;
        bytes32 adminRole;
    }

    // 主要数据结构
    mapping(bytes32 => RoleData) private _roles;

    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;

   
    modifier onlyRole(bytes32 role) {
        _checkRole(role);
        _;
    }

    
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == type(IAccessControl).interfaceId || super.supportsInterface(interfaceId);
    }

    function hasRole(bytes32 role, address account) public view virtual override returns (bool) {
        return _roles[role].members[account];
    }

    function _checkRole(bytes32 role) internal view virtual {
        _checkRole(role, _msgSender());
    }

    function _checkRole(bytes32 role, address account) internal view virtual {
        if (!hasRole(role, account)) {
            revert(
                string(
                    abi.encodePacked(
                        "AccessControl: account ",
                        Strings.toHexString(account),
                        " is missing role ",
                        Strings.toHexString(uint256(role), 32)
                    )
                )
            );
        }
    }

    function getRoleAdmin(bytes32 role) public view virtual override returns (bytes32) {
        return _roles[role].adminRole;
    }

    // 作用 : 将account的地址加入到role角色中
    // getRoleAdmin(role) : 获取role角色的ADMIN角色
    // onlyRole(getRoleAdmin(role)) : 判断msg.sender是否有是role的ADMIN角色
    function grantRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
        _grantRole(role, account);
    }

    function revokeRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }

    function renounceRole(bytes32 role, address account) public virtual override {
        require(account == _msgSender(), "AccessControl: can only renounce roles for self");
        _revokeRole(role, account);
    }

    function _setupRole(bytes32 role, address account) internal virtual {
        _grantRole(role, account);
    }

    function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal virtual {
        bytes32 previousAdminRole = getRoleAdmin(role);
        _roles[role].adminRole = adminRole;
        emit RoleAdminChanged(role, previousAdminRole, adminRole);
    }

    function _grantRole(bytes32 role, address account) internal virtual {
        if (!hasRole(role, account)) {
            _roles[role].members[account] = true;
            emit RoleGranted(role, account, _msgSender());
        }
    }

    function _revokeRole(bytes32 role, address account) internal virtual {
        if (hasRole(role, account)) {
            _roles[role].members[account] = false;
            emit RoleRevoked(role, account, _msgSender());
        }
    }
}
```

## AccessControlEnumerable.sol

AccessControl.sol的可升级版本，具体方法和AccessControl一样。

## Ownable2Step.sol

与`Ownable.sol`的区别是这个合约实现了设置合约拥有者的两阶段设置的流程。

- 调用`transferOwnership`设置`_pendingOwner`为msg.sender, 此步骤的意义在于`预设置Owner,但是实际上没有设置owner`
- 调用`acceptOwnership`判断msg.sender与第一个阶段的`_pendingOwner`是否一致，若一致则将msg.sender设置成owner。

