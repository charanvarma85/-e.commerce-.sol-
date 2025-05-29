// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/**
 * @title E-Commerce Marketplace Smart Contract
 * @dev A decentralized marketplace for buying and selling products
 */
contract Project {
    
    // State variables
    address public owner;
    uint256 public productCounter;
    uint256 public constant PLATFORM_FEE_PERCENT = 2; // 2% platform fee
    
    // Structs
    struct Product {
        uint256 id;
        string name;
        string description;
        uint256 price;
        address payable seller;
        bool isActive;
        uint256 stock;
        string imageHash; // IPFS hash for product image
    }
    
    struct Order {
        uint256 orderId;
        uint256 productId;
        address buyer;
        uint256 quantity;
        uint256 totalAmount;
        bool isDelivered;
        uint256 timestamp;
    }
    
    // Mappings
    mapping(uint256 => Product) public products;
    mapping(uint256 => Order) public orders;
    mapping(address => uint256[]) public sellerProducts;
    mapping(address => uint256[]) public buyerOrders;
    
    // Counters
    uint256 public orderCounter;
    
    // Events
    event ProductListed(
        uint256 indexed productId,
        string name,
        uint256 price,
        address indexed seller
    );
    
    event ProductPurchased(
        uint256 indexed orderId,
        uint256 indexed productId,
        address indexed buyer,
        uint256 quantity,
        uint256 totalAmount
    );
    
    event OrderDelivered(
        uint256 indexed orderId,
        address indexed buyer
    );
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
    
    modifier onlyActiveSeller(uint256 _productId) {
        require(products[_productId].seller == msg.sender, "Not the seller");
        require(products[_productId].isActive, "Product not active");
        _;
    }
    
    constructor() {
        owner = msg.sender;
        productCounter = 0;
        orderCounter = 0;
    }
    
    /**
     * @dev Core Function 1: List a new product on the marketplace
     * @param _name Product name
     * @param _description Product description
     * @param _price Product price in wei
     * @param _stock Initial stock quantity
     * @param _imageHash IPFS hash of product image
     */
    function listProduct(
        string memory _name,
        string memory _description,
        uint256 _price,
        uint256 _stock,
        string memory _imageHash
    ) public returns (uint256) {
        require(bytes(_name).length > 0, "Product name cannot be empty");
        require(_price > 0, "Price must be greater than 0");
        require(_stock > 0, "Stock must be greater than 0");
        
        productCounter++;
        
        products[productCounter] = Product({
            id: productCounter,
            name: _name,
            description: _description,
            price: _price,
            seller: payable(msg.sender),
            isActive: true,
            stock: _stock,
            imageHash: _imageHash
        });
        
        sellerProducts[msg.sender].push(productCounter);
        
        emit ProductListed(productCounter, _name, _price, msg.sender);
        
        return productCounter;
    }
    
    /**
     * @dev Core Function 2: Purchase a product from the marketplace
     * @param _productId ID of the product to purchase
     * @param _quantity Quantity to purchase
     */
    function purchaseProduct(uint256 _productId, uint256 _quantity) public payable {
        require(_productId > 0 && _productId <= productCounter, "Invalid product ID");
        require(_quantity > 0, "Quantity must be greater than 0");
        
        Product storage product = products[_productId];
        require(product.isActive, "Product is not active");
        require(product.stock >= _quantity, "Insufficient stock");
        require(product.seller != msg.sender, "Cannot buy your own product");
        
        uint256 totalCost = product.price * _quantity;
        require(msg.value >= totalCost, "Insufficient payment");
        
        // Calculate platform fee
        uint256 platformFee = (totalCost * PLATFORM_FEE_PERCENT) / 100;
        uint256 sellerAmount = totalCost - platformFee;
        
        // Update product stock
        product.stock -= _quantity;
        if (product.stock == 0) {
            product.isActive = false;
        }
        
        // Create order
        orderCounter++;
        orders[orderCounter] = Order({
            orderId: orderCounter,
            productId: _productId,
            buyer: msg.sender,
            quantity: _quantity,
            totalAmount: totalCost,
            isDelivered: false,
            timestamp: block.timestamp
        });
        
        buyerOrders[msg.sender].push(orderCounter);
        
        // Transfer payments
        product.seller.transfer(sellerAmount);
        payable(owner).transfer(platformFee);
        
        // Refund excess payment
        if (msg.value > totalCost) {
            payable(msg.sender).transfer(msg.value - totalCost);
        }
        
        emit ProductPurchased(orderCounter, _productId, msg.sender, _quantity, totalCost);
    }
    
    /**
     * @dev Core Function 3: Mark order as delivered (seller only)
     * @param _orderId ID of the order to mark as delivered
     */
    function markAsDelivered(uint256 _orderId) public {
        require(_orderId > 0 && _orderId <= orderCounter, "Invalid order ID");
        
        Order storage order = orders[_orderId];
        Product storage product = products[order.productId];
        
        require(product.seller == msg.sender, "Only seller can mark as delivered");
        require(!order.isDelivered, "Order already marked as delivered");
        
        order.isDelivered = true;
        
        emit OrderDelivered(_orderId, order.buyer);
    }
    
    // View functions
    function getProduct(uint256 _productId) public view returns (
        uint256 id,
        string memory name,
        string memory description,
        uint256 price,
        address seller,
        bool isActive,
        uint256 stock,
        string memory imageHash
    ) {
        require(_productId > 0 && _productId <= productCounter, "Invalid product ID");
        Product memory product = products[_productId];
        return (
            product.id,
            product.name,
            product.description,
            product.price,
            product.seller,
            product.isActive,
            product.stock,
            product.imageHash
        );
    }
    
    function getOrder(uint256 _orderId) public view returns (
        uint256 orderId,
        uint256 productId,
        address buyer,
        uint256 quantity,
        uint256 totalAmount,
        bool isDelivered,
        uint256 timestamp
    ) {
        require(_orderId > 0 && _orderId <= orderCounter, "Invalid order ID");
        Order memory order = orders[_orderId];
        return (
            order.orderId,
            order.productId,
            order.buyer,
            order.quantity,
            order.totalAmount,
            order.isDelivered,
            order.timestamp
        );
    }
    
    function getSellerProducts(address _seller) public view returns (uint256[] memory) {
        return sellerProducts[_seller];
    }
    
    function getBuyerOrders(address _buyer) public view returns (uint256[] memory) {
        return buyerOrders[_buyer];
    }
    
    // Admin functions
    function updatePlatformOwner(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), "Invalid address");
        owner = _newOwner;
    }
    
    function deactivateProduct(uint256 _productId) public onlyActiveSeller(_productId) {
        products[_productId].isActive = false;
    }
    
    function updateProductStock(uint256 _productId, uint256 _newStock) public onlyActiveSeller(_productId) {
        products[_productId].stock = _newStock;
        if (_newStock > 0 && !products[_productId].isActive) {
            products[_productId].isActive = true;
        }
    }
}# -e.commerce-.sol-
![image](https://github.com/user-attachments/assets/97161571-3765-434d-ba3c-d111662d4e04)
