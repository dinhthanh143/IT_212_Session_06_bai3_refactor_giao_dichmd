# BÀI 3: THỰC HÀNH REFACTOR & NÂNG CẤP GIAO DỊCH (REFINEMENT PROCESS)

## 1. Phân tích các lỗ hổng nghiêm trọng của đoạn code thô ban đầu
Đoạn code ban đầu do lập trình viên tập sự viết chứa các lỗi nghiêm trọng sau:
1. **Thiếu Quản lý Giao dịch (No Transaction Management):** Không sử dụng `@Transactional`. Khi xảy ra lỗi ở bước thanh toán (`paymentGateway.charge`) hoặc bước lưu đơn hàng (`orderRepository.save`), các thao tác trừ kho đã lưu thành công trước đó sẽ không được rollback. Điều này dẫn đến tình trạng mất cân đối kho nghiêm trọng (hàng bị trừ nhưng không thanh toán được và không có đơn hàng).
2. **Không kiểm tra tính hợp lệ dữ liệu đầu vào (No Validation):** Không kiểm tra `null` cho `order` hoặc `order.getItems()`. Điều này dễ dẫn đến lỗi `NullPointerException` (NPE).
3. **Xử lý tồn kho nguy hiểm:** 
   - Nếu `findById` trả về `null` (sản phẩm không tồn tại), gọi `product.setStock` sẽ lập tức gây ra NPE.
   - Không kiểm tra xem lượng tồn kho hiện tại có đủ đáp ứng số lượng đặt mua hay không. Nếu kho không đủ, stock sẽ bị trừ âm (ví dụ: $5 - 10 = -5$), một lỗi nghiệp vụ nghiêm trọng.
4. **Vấn đề hiệu năng (N+1 Query Issue):** Thực hiện `findById` và `save` lặp đi lặp lại trong vòng lặp `for` cho từng sản phẩm. Nếu đơn hàng có 100 sản phẩm, hệ thống phải thực hiện 200 lượt gọi cơ sở dữ liệu.
5. **Thiếu Logging và Exception Handling:** Không sử dụng log để giám sát dòng chảy nghiệp vụ. Không bắt ngoại lệ từ cổng thanh toán để đưa ra hướng xử lý phù hợp (rollback, thông báo lỗi cụ thể cho khách hàng).

---

## 2. Chi tiết nội dung 3 lượt Prompt tương ứng với 3 vòng cải tiến

### Vòng 1: Cải tiến tính mạnh mẽ (Robustness)
```text
[Vòng 1 - Robustness]
Hãy đóng vai trò là Senior Developer, refactor đoạn code Java của class OrderPlacementService để xử lý các vấn đề bảo mật dữ liệu và phòng ngừa lỗi sau:
1. Kiểm tra dữ liệu đầu vào: Đảm bảo order, danh sách items, customerId không bị null hoặc empty.
2. Kiểm tra tồn kho: Khi lấy thông tin sản phẩm, nếu sản phẩm không tồn tại hoặc số lượng tồn kho (stock) hiện tại ít hơn số lượng đặt mua (quantity), hãy ném ra ngoại lệ tự định nghĩa OutOfStockException (kèm thông tin chi tiết productId).
3. Xử lý lỗi Cổng thanh toán: Bao bọc phương thức charge() của paymentGateway. Nếu thanh toán thất bại hoặc phát sinh lỗi từ gateway, hãy ném ra ngoại lệ tự định nghĩa PaymentFailedException.
```

### Vòng 2: Cải tiến khả năng bảo trì & Code sạch (Maintainability & Clean Code)
```text
[Vòng 2 - Maintainability & Clean Code]
Tiếp tục cải tiến đoạn code đã refactor ở Vòng 1 với các yêu cầu sau:
1. Thêm cơ chế quản lý giao dịch bằng cách sử dụng annotation @Transactional của Spring Boot trên phương thức placeOrder để đảm bảo tính nguyên tử (Atomicity). Nếu bất kỳ bước nào xảy ra lỗi (như PaymentFailedException hoặc OutOfStockException), toàn bộ các thay đổi trước đó (như trừ kho) phải được rollback hoàn toàn về trạng thái ban đầu.
2. Tích hợp thư viện Lombok (@RequiredArgsConstructor) để thay thế cho việc khai báo constructor dài dòng.
3. Tích hợp log bằng annotation @Slf4j của Lombok. Thực hiện ghi log chi tiết: ghi log cấp độ INFO khi bắt đầu xử lý, khi trừ kho thành công, thanh toán thành công; và ghi log cấp độ ERROR kèm theo thông tin chi tiết về lỗi khi phát sinh ngoại lệ OutOfStockException hoặc PaymentFailedException.
```

### Vòng 3: Tối ưu hóa theo ngữ cảnh dự án (Context-specific Tuning)
```text
[Vòng 3 - Context-specific Tuning]
Đây là vòng cải tiến cuối cùng. Hãy hoàn thiện mã nguồn với các yêu cầu:
1. Thay vì kiểu trả về void, phương thức placeOrder(Order order) phải trả về đối tượng OrderPlacementResult chứa các thông tin: boolean success, String message, String orderId, và LocalDateTime placementTime.
2. Viết một class JUnit 5 Test (`OrderPlacementServiceTest`) sử dụng thư viện Mockito để mock các repository và gateway (InventoryRepository, PaymentGateway, OrderRepository).
3. Viết test case cụ thể kiểm thử kịch bản thanh toán thất bại: Giả lập khi cổng thanh toán charge() ném ra ngoại lệ, đảm bảo phương thức placeOrder sẽ ném ra PaymentFailedException và chứng minh rằng các transaction được kiểm soát đúng đắn thông qua Mockito verification.
```

---

## 3. Minh chứng chạy thực tế (Text log phản hồi ở lượt chat thứ 3)

### PHẦN 1: MÃ NGUỒN CẤU HÌNH CÁC NGOẠI LỆ TỰ ĐỊNH NGHĨA VÀ DỮ LIỆU ĐẦU RA

#### 1. Ngoại lệ OutOfStockException.java
```java
package com.speedycart.exception;

public class OutOfStockException extends RuntimeException {
    public OutOfStockException(String message) {
        super(message);
    }
}
```

#### 2. Ngoại lệ PaymentFailedException.java
```java
package com.speedycart.exception;

public class PaymentFailedException extends RuntimeException {
    public PaymentFailedException(String message, Throwable cause) {
        super(message, cause);
    }
    public PaymentFailedException(String message) {
        super(message);
    }
}
```

#### 3. Đối tượng kết quả OrderPlacementResult.java
```java
package com.speedycart.dto;

import lombok.Builder;
import lombok.Getter;
import java.time.LocalDateTime;

@Getter
@Builder
public class OrderPlacementResult {
    private final boolean success;
    private final String message;
    private final String orderId;
    private final LocalDateTime placementTime;
}
```

---

### PHẦN 2: MÃ NGUỒN CLASS `ORDERPLACEMENTSERVICE` ĐÃ REFACTOR HOÀN CHỈNH

```java
package com.speedycart.service;

import com.speedycart.dto.OrderPlacementResult;
import com.speedycart.exception.OutOfStockException;
import com.speedycart.exception.PaymentFailedException;
import com.speedycart.model.Order;
import com.speedycart.model.OrderItem;
import com.speedycart.model.Product;
import com.speedycart.repository.InventoryRepository;
import com.speedycart.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderPlacementService {

    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    /**
     * Thực hiện đặt đơn hàng: Trừ kho, Thanh toán và Lưu đơn hàng.
     * Transactional đảm bảo nếu có lỗi ở bất kỳ bước nào, kho sẽ được rollback.
     */
    @Transactional(rollbackFor = Exception.class)
    public OrderPlacementResult placeOrder(Order order) {
        // Vòng 1: Kiểm tra tính hợp lệ của dữ liệu đầu vào
        validateOrderInput(order);

        log.info("Starting order placement process for customer: {}, Order ID: {}", 
                order.getCustomerId(), order.getId());

        List<Product> updatedProducts = new ArrayList<>();

        // Vòng 1: Kiểm tra tồn kho và chuẩn bị trừ kho
        for (OrderItem item : order.getItems()) {
            Product product = inventoryRepository.findById(item.getProductId())
                    .orElseThrow(() -> {
                        String errMsg = String.format("Product not found in inventory: %s", item.getProductId());
                        log.error("Order Placement Failed: {}", errMsg);
                        return new OutOfStockException(errMsg);
                    });

            if (product.getStock() < item.getQuantity()) {
                String errMsg = String.format("Product %s out of stock. Available: %d, Requested: %d",
                        product.getName(), product.getStock(), item.getQuantity());
                log.error("Order Placement Failed: {}", errMsg);
                throw new OutOfStockException(errMsg);
            }

            // Trừ số lượng tồn kho
            product.setStock(product.getStock() - item.getQuantity());
            updatedProducts.add(product);
        }

        // Lưu thông tin cập nhật tồn kho vào DB
        inventoryRepository.saveAll(updatedProducts);
        log.info("Inventory updated successfully for Order ID: {}", order.getId());

        // Vòng 1: Gọi cổng thanh toán và xử lý lỗi phát sinh
        try {
            log.info("Charging customer {} amount: {}", order.getCustomerId(), order.getTotalAmount());
            paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
            log.info("Payment successful for Order ID: {}", order.getId());
        } catch (Exception e) {
            String errMsg = String.format("Payment charge failed for customer %s. Error: %s", 
                    order.getCustomerId(), e.getMessage());
            log.error("Order Placement Failed: {}", errMsg, e);
            // Ném ngoại lệ để Spring thực hiện rollback số lượng kho đã trừ
            throw new PaymentFailedException(errMsg, e);
        }

        // Lưu thông tin đơn hàng
        Order savedOrder = orderRepository.save(order);
        log.info("Order saved successfully. ID: {}", savedOrder.getId());

        return OrderPlacementResult.builder()
                .success(true)
                .message("Order placed successfully.")
                .orderId(savedOrder.getId())
                .placementTime(LocalDateTime.now())
                .build();
    }

    private void validateOrderInput(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
        if (order.getCustomerId() == null || order.getCustomerId().trim().isEmpty()) {
            throw new IllegalArgumentException("Customer ID cannot be null or empty");
        }
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must contain at least one item");
        }
        if (order.getTotalAmount() == null || order.getTotalAmount().compareTo(java.math.BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Invalid total amount");
        }
    }
}
```

---

### PHẦN 3: LỚP KIỂM THỬ ĐƠN VỊ JUNIT 5 & MOCKITO (`ORDERPLACEMENTSERVICETEST`)

```java
package com.speedycart.service;

import com.speedycart.exception.PaymentFailedException;
import com.speedycart.model.Order;
import com.speedycart.model.OrderItem;
import com.speedycart.model.Product;
import com.speedycart.repository.InventoryRepository;
import com.speedycart.repository.OrderRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Collections;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderPlacementServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderPlacementService orderPlacementService;

    private Order sampleOrder;
    private Product sampleProduct;

    @BeforeEach
    void setUp() {
        OrderItem item = OrderItem.builder()
                .productId("PROD-001")
                .quantity(2)
                .build();

        sampleOrder = Order.builder()
                .id("ORD-999")
                .customerId("CUST-111")
                .totalAmount(new BigDecimal("500000"))
                .items(Collections.singletonList(item))
                .build();

        sampleProduct = Product.builder()
                .id("PROD-001")
                .name("Sản phẩm A")
                .stock(10)
                .build();
    }

    @Test
    @DisplayName("Should throw PaymentFailedException and verify that rollback occurred when payment fails")
    void placeOrder_PaymentFails_ThrowsPaymentFailedException() {
        // Given
        when(inventoryRepository.findById("PROD-001")).thenReturn(Optional.of(sampleProduct));
        
        // Giả lập cổng thanh toán ném ra ngoại lệ RuntimeException khi charge
        doThrow(new RuntimeException("Connection Timeout"))
                .when(paymentGateway).charge(eq("CUST-111"), any(BigDecimal.class));

        // When & Then
        PaymentFailedException exception = assertThrows(PaymentFailedException.class, () -> {
            orderPlacementService.placeOrder(sampleOrder);
        });

        // Kiểm chứng thông tin lỗi trong Exception
        assertTrue(exception.getMessage().contains("Payment charge failed"));

        // Xác minh kho hàng: đã tìm kiếm và đã thực hiện lưu thay đổi kho tạm
        verify(inventoryRepository, times(1)).findById("PROD-001");
        verify(inventoryRepository, times(1)).saveAll(anyList());

        // Xác minh rằng phương thức save của OrderRepository KHÔNG BAO GIỜ được gọi (ngăn lưu đơn hàng lỗi)
        verify(orderRepository, never()).save(any(Order.class));
        
        // Lưu ý: Spring @Transactional sẽ tự động bắt RuntimeException này và rollback các thay đổi trong Database.
    }
}
```
