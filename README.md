===== 【機能①】取引一覧画面 フィルター機能 =====

1. TradeController.java
--------------------------------------------------
@Controller
public class TradeController {

    @Autowired
    private TradeRepository tradeRepository;

    @Autowired
    private StockRepository stockRepository;

    @GetMapping("/trade")
    public String tradeList(@RequestParam(required = false) String ticker,
                            @RequestParam(defaultValue = "all") String date,
                            Model model) {
        List<Trade> trades;

        // フィルター条件なし：全件取得
        if ((ticker == null || ticker.isEmpty()) && date.equals("all")) {
            trades = tradeRepository.findAll();
        }
        // 日付 = today（当日のみ取得）
        else if (date.equals("today")) {
            LocalDate today = LocalDate.now();
            LocalDateTime start = today.atStartOfDay();
            LocalDateTime end = today.atTime(LocalTime.MAX);

            if (ticker == null || ticker.isEmpty()) {
                trades = tradeRepository.findByTradedDatetimeBetween(start, end);
            } else {
                trades = tradeRepository.findByTickerAndTradedDatetimeBetween(ticker, start, end);
            }
        }
        // Ticker のみで検索
        else {
            trades = tradeRepository.findByTickerContaining(ticker);
        }

        model.addAttribute("trades", trades);
        return "trade";
    }
}
--------------------------------------------------

2. TradeRepository.java
--------------------------------------------------
public interface TradeRepository extends JpaRepository<Trade, Integer> {

    List<Trade> findByTradedDatetimeBetween(LocalDateTime start, LocalDateTime end); // ★追加

    @Query("SELECT t FROM Trade t WHERE t.stock.ticker = :ticker AND t.tradedDatetime BETWEEN :start AND :end")
    List<Trade> findByTickerAndTradedDatetimeBetween(@Param("ticker") String ticker,
                                                      @Param("start") LocalDateTime start,
                                                      @Param("end") LocalDateTime end); // ★追加

    @Query("SELECT t FROM Trade t WHERE t.stock.ticker LIKE %:ticker%")
    List<Trade> findByTickerContaining(@Param("ticker") String ticker); // ★追加
}
--------------------------------------------------

3. trade.html（HTML テンプレート冒頭に追加）
--------------------------------------------------
<form method="get" action="/trade">
  <input type="text" name="ticker" placeholder="Ticker（任意）"/>
  <label><input type="radio" name="date" value="all" checked> All</label>
  <label><input type="radio" name="date" value="today"> Today</label>
  <button type="submit" style="background-color:#ccffcc;">フィルター</button>
</form>
--------------------------------------------------


===== 【機能②】取引入力 validation追加 =====

1. WithinIssuedLimit.java（annotation/ 配下に新規作成）
--------------------------------------------------
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = WithinIssuedLimitValidator.class)
public @interface WithinIssuedLimit {
    String message() default "保有数と入力数の合計が発行済株式数を超えています。";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
--------------------------------------------------

2. WithinIssuedLimitValidator.java
--------------------------------------------------
public class WithinIssuedLimitValidator implements ConstraintValidator<WithinIssuedLimit, TradeInputDto> {

    @Autowired
    private StockRepository stockRepository;

    @Autowired
    private TradeRepository tradeRepository;

    @Override
    public boolean isValid(TradeInputDto dto, ConstraintValidatorContext context) {
        Stock stock = stockRepository.findByTicker(dto.getTicker());
        if (stock == null) return true;

        Long held = tradeRepository.getHoldingQuantityByTicker(dto.getTicker());
        if (held == null) held = 0L;

        return held + dto.getQuantity() <= stock.getSharesIssued();
    }
}
--------------------------------------------------

3. TradeRepository.java（機能①に追加でさらに以下を追加）
--------------------------------------------------
@Query("SELECT COALESCE(SUM(CASE WHEN t.side = 'BUY' THEN t.quantity ELSE -t.quantity END), 0) " +
       "FROM Trade t WHERE t.stock.ticker = :ticker")
Long getHoldingQuantityByTicker(@Param("ticker") String ticker); // ★追加
--------------------------------------------------

4. TradeInputDto.java（クラスにアノテーション追加）
--------------------------------------------------
@WithinIssuedLimit // ★追加：クラスの上に追加
public class TradeInputDto {

    @NotNull
    private String ticker;

    @NotNull
    private LocalDateTime tradedDatetime;

    @NotNull
    private String side;

    @NotNull
    private Long quantity;

    @NotNull
    private BigDecimal tradedPrice;

    // getter/setter...
}
--------------------------------------------------


===== 【機能③】時価一括登録画面（/marketprice/bulk） =====

1. HTML（templates/marketprice_bulk.html）
--------------------------------------------------
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Market Price Bulk Register</title>
</head>
<body>
<h1>Market Price Register</h1>

<form method="post" action="/marketprice/bulk">
    <label for="bulkText">Market Price</label><br/>
    <textarea id="bulkText" name="bulkText" rows="10" cols="50" placeholder="2432,1500.00
4523,1200.50
..."></textarea><br/><br/>
    <button type="submit">Register</button>
</form>

<div th:if="${error}" style="color:red;" th:text="${error}"></div>
</body>
</html>
--------------------------------------------------

2. MarketPriceController.java（新規作成）
--------------------------------------------------
@Controller
public class MarketPriceController {

    @Autowired
    private StockRepository stockRepository;

    @Autowired
    private MarketPriceRepository marketPriceRepository;

    @GetMapping("/marketprice/bulk")
    public String showBulkForm() {
        return "marketprice_bulk";
    }

    @PostMapping("/marketprice/bulk")
    public String registerMarketPrices(@RequestParam String bulkText, Model model) {
        if (bulkText == null || bulkText.isBlank()) {
            model.addAttribute("error", "入力が空です。");
            return "marketprice_bulk";
        }

        String[] lines = bulkText.split("\r?\n");
        Set<String> seenTickers = new HashSet<>();
        List<MarketPrice> marketPrices = new ArrayList<>();

        for (String line : lines) {
            String[] parts = line.trim().split(",");
            if (parts.length != 2) {
                model.addAttribute("error", "列数が正しくありません: " + line);
                return "marketprice_bulk";
            }

            String ticker = parts[0].trim();
            String priceStr = parts[1].trim();

            if (ticker.isEmpty() || seenTickers.contains(ticker)) {
                model.addAttribute("error", "空のtickerまたは重複があります: " + ticker);
                return "marketprice_bulk";
            }

            seenTickers.add(ticker);

            BigDecimal price;
            try {
                price = new BigDecimal(priceStr);
                if (price.scale() > 2 || price.compareTo(BigDecimal.ZERO) < 0) {
                    throw new NumberFormatException();
                }
            } catch (NumberFormatException e) {
                model.addAttribute("error", "不正な価格形式です: " + priceStr);
                return "marketprice_bulk";
            }

            Stock stock = stockRepository.findByTicker(ticker);
            if (stock == null) {
                model.addAttribute("error", "存在しないtickerです: " + ticker);
                return "marketprice_bulk";
            }

            MarketPrice mp = new MarketPrice();
            mp.setStock(stock);
            mp.setMarketPrice(price);
            mp.setCreatedDatetime(LocalDateTime.now());

            marketPrices.add(mp);
        }

        marketPriceRepository.saveAll(marketPrices);
        return "redirect:/positions";
    }
}
--------------------------------------------------

3. MarketPrice.java（model 配下に新規作成）
--------------------------------------------------
@Entity
public class MarketPrice {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @ManyToOne
    @JoinColumn(name = "stock_id", nullable = false)
    private Stock stock;

    @Column(name = "market_price", nullable = false, precision = 10, scale = 2)
    private BigDecimal marketPrice;

    @Column(name = "created_datetime", nullable = false)
    private LocalDateTime createdDatetime = LocalDateTime.now();

    // getter/setter
    public Integer getId() { return id; }
    public Stock getStock() { return stock; }
    public void setStock(Stock stock) { this.stock = stock; }
    public BigDecimal getMarketPrice() { return marketPrice; }
    public void setMarketPrice(BigDecimal marketPrice) { this.marketPrice = marketPrice; }
    public LocalDateTime getCreatedDatetime() { return createdDatetime; }
    public void setCreatedDatetime(LocalDateTime createdDatetime) { this.createdDatetime = createdDatetime; }
}
--------------------------------------------------

4. MarketPriceRepository.java（新規作成）
--------------------------------------------------
public interface MarketPriceRepository extends JpaRepository<MarketPrice, Integer> {

    @Query("SELECT m FROM MarketPrice m WHERE m.stock.id = :stockId ORDER BY m.createdDatetime DESC")
    List<MarketPrice> findLatestByStockId(@Param("stockId") Integer stockId);
}
--------------------------------------------------

5. StockRepository.java にメソッド追加（既に存在していれば不要）
--------------------------------------------------
Stock findByTicker(String ticker);
--------------------------------------------------


===== src/main/java/controller/PositionController.java =====

@Controller
public class PositionController {

    @Autowired
    private PositionService positionService;

    @GetMapping("/positions")
    public String showPositions(Model model) {
        model.addAttribute("positions", positionService.calculatePositions());
        return "positions";
    }
}


===== src/main/java/service/PositionService.java =====

@Service
public class PositionService {

    @Autowired
    private TradeRepository tradeRepository;

    @Autowired
    private MarketPriceRepository marketPriceRepository;

    public List<PositionDto> calculatePositions() {
        List<Trade> trades = tradeRepository.findAllWithStock();
        Map<Stock, List<Trade>> grouped = trades.stream().collect(Collectors.groupingBy(Trade::getStock));

        List<PositionDto> result = new ArrayList<>();

        for (Stock stock : grouped.keySet()) {
            List<Trade> stockTrades = grouped.get(stock);
            PositionDto dto = new PositionDto(stock.getTicker(), stock.getName());

            long quantity = 0;
            BigDecimal totalBuy = BigDecimal.ZERO;
            long totalBuyQty = 0;
            BigDecimal realizedProfit = BigDecimal.ZERO;

            for (Trade t : stockTrades) {
                if (t.getSide() == Side.BUY) {
                    quantity += t.getQuantity();
                    totalBuy = totalBuy.add(t.getTradedPrice().multiply(BigDecimal.valueOf(t.getQuantity())));
                    totalBuyQty += t.getQuantity();
                } else {
                    quantity -= t.getQuantity();
                    if (totalBuyQty > 0) {
                        BigDecimal avg = totalBuy.divide(BigDecimal.valueOf(totalBuyQty), 2, RoundingMode.HALF_UP);
                        BigDecimal profit = (t.getTradedPrice().subtract(avg))
                                .multiply(BigDecimal.valueOf(t.getQuantity()));
                        realizedProfit = realizedProfit.add(profit);
                    }
                }
            }

            dto.quantity = quantity;
            dto.avgPrice = (totalBuyQty > 0) ?
                totalBuy.divide(BigDecimal.valueOf(totalBuyQty), 2, RoundingMode.HALF_UP) : null;
            dto.realizedProfit = (realizedProfit.compareTo(BigDecimal.ZERO) == 0) ? null : realizedProfit;

            MarketPrice mp = marketPriceRepository.findLatestByStockId(stock.getId());
            if (mp != null) {
                dto.marketPrice = mp.getMarketPrice();
                if (quantity > 0 && dto.avgPrice != null) {
                    dto.valuation = mp.getMarketPrice().multiply(BigDecimal.valueOf(quantity)).setScale(2, RoundingMode.HALF_UP);
                    dto.unrealizedProfit = (mp.getMarketPrice().subtract(dto.avgPrice))
                            .multiply(BigDecimal.valueOf(quantity)).setScale(2, RoundingMode.HALF_UP);
                }
            }

            result.add(dto);
        }

        result.sort(Comparator.comparing(p -> p.ticker));
        return result;
    }
}


===== src/main/java/dto/PositionDto.java =====

public class PositionDto {
    public String ticker;
    public String name;
    public long quantity;
    public BigDecimal avgPrice;
    public BigDecimal realizedProfit;
    public BigDecimal marketPrice;
    public BigDecimal valuation;
    public BigDecimal unrealizedProfit;

    public PositionDto(String ticker, String name) {
        this.ticker = ticker;
        this.name = name;
    }
}


===== src/main/java/repository/TradeRepository.java =====

@Query("SELECT t FROM Trade t JOIN FETCH t.stock")
List<Trade> findAllWithStock();


===== src/main/java/repository/MarketPriceRepository.java =====

@Query("SELECT m FROM MarketPrice m WHERE m.stock.id = :stockId ORDER BY m.createdDatetime DESC")
List<MarketPrice> findLatestByStockId(@Param("stockId") Integer stockId);


===== src/main/resources/templates/positions.html =====

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Positions</title>
    <style>
        th { text-align: center; }
        td.right { text-align: right; }
        td.left { text-align: left; }
        .red { color: red; }
        .green { color: green; }
        .gray { color: gray; }
    </style>
</head>
<body>
<h1>Positions</h1>
<table border="1">
    <thead>
        <tr>
            <th>Ticker</th>
            <th>Name</th>
            <th>Quantity</th>
            <th>Average Unit Price</th>
            <th>Realized P/L</th>
            <th>Market Price</th>
            <th>Valuation</th>
            <th>Unrealized P/L</th>
        </tr>
    </thead>
    <tbody>
        <tr th:each="pos : ${positions}">
            <td class="left" th:text="${pos.ticker}"></td>
            <td class="left" th:text="${pos.name}"></td>
            <td class="right" th:text="${pos.quantity}"></td>
            <td class="right" th:text="${pos.avgPrice != null ? pos.avgPrice : 'N/A'}"></td>
            <td class="right"
                th:classappend="${pos.realizedProfit > 0 ? 'red' : (pos.realizedProfit < 0 ? 'green' : 'gray')}"
                th:text="${pos.realizedProfit != null ? pos.realizedProfit : 'N/A'}"></td>
            <td class="right" th:text="${pos.marketPrice != null ? pos.marketPrice : 'N/A'}"></td>
            <td class="right"
                th:classappend="${pos.valuation > 0 ? 'red' : (pos.valuation == 0 ? 'gray' : '')}"
                th:text="${pos.valuation != null ? pos.valuation : 'N/A'}"></td>
            <td class="right"
                th:classappend="${pos.unrealizedProfit > 0 ? 'red' : (pos.unrealizedProfit < 0 ? 'green' : 'gray')}"
                th:text="${pos.unrealizedProfit != null ? pos.unrealizedProfit : 'N/A'}"></td>
        </tr>
    </tbody>
</table>
</body>
</html>

