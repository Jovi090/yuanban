package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.view.MenuView;

public class MainController {
    private final MenuView menuView;
    private final StockController stockController;
    private final TradeController tradeController;

    public MainController(MenuView menuView, StockController stockController, TradeController tradeController) {
        this.menuView = menuView;
        this.stockController = stockController;
        this.tradeController = tradeController;
    }

    public void start() {
        boolean running = true;
        while (running) {
            String choice = menuView.displayMainMenu();
            switch (choice) {
                case "1" ->{
                    System.out.println("「銘柄マスタ一覧表示」が選択されました。");
                    stockController.displayAllStocks();}
                case "2" ->{
                    System.out.println("「銘柄マスタ新規登録」が選択されました。");
                    stockController.addNewStock();}
                case "3" ->{
                    System.out.println("「取引新規登録」が選択されました。");
                    tradeController.recordNewTrade();}
                case "4" ->{
                    System.out.println("「取引一覧表示」が選択されました。");
                    tradeController.displayAllTrades();}
                case "5" -> {
                    System.out.println("「保有ポジション表示」が選択されました。");
                    tradeController.displayHoldings();}
                case "9" -> {
                    System.out.println("アプリケーションを終了します。");
                    running = false;
                }
                default ->  System.out.println("対応するメニューは存在しません。");
            }
            System.out.println("---");
        }
        menuView.close();
    }
}

package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Market;
import simplex.bn25.zhao335952.trading.model.Stock;
import simplex.bn25.zhao335952.trading.repository.StockRepository;
import simplex.bn25.zhao335952.trading.view.StockView;

import java.util.List;
import java.util.Scanner;

public class StockController {
    private final StockRepository stockRepository;
    private final StockView stockView;
    private final Scanner scanner = new Scanner(System.in);

    public StockController(StockRepository stockRepository, StockView stockView) {
        this.stockRepository = stockRepository;
        this.stockView = stockView;
    }

    public void displayAllStocks() {
        List<Stock> stocks = stockRepository.getAllStocks();
        stockView.displayStockList(stocks);
    }

    public void addNewStock() {
        String ticker = inputTicker();
        String productName = inputProductName();
        String market = inputMarket();
        long sharesIssued = inputSharesIssued();

        Stock stock = new Stock(ticker, productName, market, sharesIssued);

        if (stockRepository.saveStock(stock)) {
            stockView.showStockAddedMessage(stock);
        } else {
            System.out.print("データの書き込みにエラーが発生しました。");
        }
    }

    private String inputTicker() {
        String ticker;
        while (true) {
            System.out.print("銘柄コードを入力してください (4桁の半角英数字): ");
            ticker = scanner.nextLine().trim();

            if (!Stock.isValidTicker(ticker)) {
                System.out.println("無効な銘柄コードです。再度入力してください。");
                continue;
            }

            if (stockRepository.isTickerRegistered(ticker)) {
                System.out.println("既に登録されている銘柄コードです。再度入力してください。");
            } else {
                break;
            }
        }
        return ticker;
    }

    private String inputProductName() {
        String productName;
        while (true) {
            System.out.print("銘柄名を入力してください: ");
            productName = scanner.nextLine().trim();
            if (Stock.isValidProductName(productName)) {
                break;
            } else {
                System.out.println("無効な銘柄名です。再度入力してください。");
            }
        }
        return productName;
    }

    private String inputMarket() {
        String market;
        while (true) {
            System.out.print("上場市場を入力してください（Prime, Standard, Growth）：");
            market = scanner.nextLine().trim();
            if (Market.isValidMarket(market)) {
                break;
            } else {
                System.out.println("無効な上場市場です。再度入力してください。");
            }
        }
        return market;
    }

    private long inputSharesIssued() {
        long sharesIssued;
        while (true) {
            System.out.print("発行済み株式数を入力してください: ");
            String input = scanner.nextLine().trim();
            if (Stock.isValidSharesIssued(input)) {
                sharesIssued = Long.parseLong(input);
                break;
            } else {
                System.out.println("無効な株式発行数です。1から999999999999の間の整数を入力してください。");
            }
        }
        return sharesIssued;
    }
}
package simplex.bn25.zhao335952.trading.controller;

import simplex.bn25.zhao335952.trading.model.Side;
import simplex.bn25.zhao335952.trading.model.Trade;
import simplex.bn25.zhao335952.trading.repository.StockRepository;
import simplex.bn25.zhao335952.trading.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.TradeView;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
import java.util.TreeMap;

public class TradeController {
    private final TradeRepository tradeRepository;
    private final StockRepository stockRepository;
    private final PositionView positionView;
    private final TradeView tradeView;
    private final Scanner scanner = new Scanner(System.in);
    private final Map<String, Integer> holdings = new TreeMap<>();
    private final Map<String, LocalDateTime> latestTradeTimes = new TreeMap<>();
    private static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public TradeController(TradeRepository tradeRepository, StockRepository stockRepository, TradeView tradeView, PositionView positionView) {
        this.tradeRepository = tradeRepository;
        this.stockRepository = stockRepository;
        this.tradeView = tradeView;
        this.positionView = positionView;
        updateSharedData();
    }

    public void displayHoldings() {
        if (holdings.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        Map<String, String> tickerToNameMap = new TreeMap<>();
        for (String ticker : holdings.keySet()) {
            String productName = stockRepository.getTickerNameByTicker(ticker);
            tickerToNameMap.put(ticker, productName);
        }
        positionView.displayHoldings(holdings, tickerToNameMap);
    }

    public void recordNewTrade() {
        System.out.print("取引日時を入力してください (yyyy-MM-dd HH:mm): ");
        LocalDateTime tradedDatetime = LocalDateTime.parse(scanner.nextLine().trim(), DATETIME_FORMATTER);

        System.out.print("銘柄コードを入力してください (4桁の半角英数字): ");
        String ticker = scanner.nextLine().trim();

        if (!tradeRepository.isTickerRegistered(ticker)) {
            System.out.println("登録されていない銘柄コードです。再度入力してください。");
            return;
        }

        System.out.print("売買区分を入力してください (Buy/Sell): ");
        Side side = Side.fromString(scanner.nextLine().trim());

        System.out.print("数量を入力してください (100株単位): ");
        int quantity = Integer.parseInt(scanner.nextLine().trim());

        System.out.print("取引単価を入力してください (小数点以下2桁まで): ");
        BigDecimal tradedUnitPrice = new BigDecimal(scanner.nextLine().trim());

        if (!isQuantityValidAfterTrade(ticker, side.getName(), quantity)) {
            System.out.println("エラー: 取引後の数量が負になります。");
            return;
        }

        if (!isTradeTimeValid(ticker, tradedDatetime)) {
            System.out.println("エラー: 取引時間が過去のデータと矛盾しています。");
            return;
        }

        LocalDateTime inputDatetime = LocalDateTime.now();
        Trade trade = new Trade(tradedDatetime, ticker, null, side.getName(), quantity, tradedUnitPrice, inputDatetime);

        if (tradeRepository.saveTrade(trade)) {
            tradeView.showTradeAddedMessage(trade);
            updateSharedData();
        } else {
            System.out.println("データの書き込みにエラーが発生しました。");
        }
    }

    public void displayAllTrades() {
        List<Trade> trades = tradeRepository.getAllTrades();
        for (Trade trade : trades) {
            String productName = stockRepository.getTickerNameByTicker(trade.getTicker());
            trade.setTickerName(productName);
        }
        tradeView.displayTradeList(trades);
    }

    private void updateSharedData() {
        holdings.clear();
        latestTradeTimes.clear();

        List<Trade> trades = tradeRepository.getAllTrades();
        for (Trade trade : trades) {
            int quantity = trade.getQuantity();
            if ("Sell".equalsIgnoreCase(trade.getSide())) {
                quantity = -quantity;
            }
            holdings.merge(trade.getTicker(), quantity, Integer::sum);

            LocalDateTime currentLatestTime = latestTradeTimes.get(trade.getTicker());
            if (currentLatestTime == null || trade.getTradedDatetime().isAfter(currentLatestTime)) {
                latestTradeTimes.put(trade.getTicker(), trade.getTradedDatetime());
            }
        }
    }

    private boolean isQuantityValidAfterTrade(String ticker, String side, int quantity) {
        int currentQuantity = holdings.getOrDefault(ticker, 0);
        int delta = "Sell".equalsIgnoreCase(side) ? -quantity : quantity;
        return currentQuantity + delta >= 0;
    }

    private boolean isTradeTimeValid(String ticker, LocalDateTime tradedDatetime) {
        LocalDateTime latestTradeTime = latestTradeTimes.get(ticker);
        return latestTradeTime == null || tradedDatetime.isAfter(latestTradeTime);
    }
}
package simplex.bn25.zhao335952.trading.model;

public enum Market {
    PRIME("P", "Prime"),
    STANDARD("S", "Standard"),
    GROWTH("G", "Growth");

    private final String symbol;
    private final String fullName;

    Market(String symbol, String fullName) {
        this.symbol = symbol;
        this.fullName = fullName;
    }

    public String getSymbol() {
        return symbol;
    }

    public String getFullName() {
        return fullName;
    }

    public static Market fromString(String market) {
        String normalizedMarket = market.trim().toLowerCase();
        return switch (normalizedMarket) {
            case "prime", "プライム", "p" -> PRIME;
            case "standard", "スタンダード", "s" -> STANDARD;
            case "growth", "グロース", "g" -> GROWTH;
            default -> throw new IllegalArgumentException("無効な上場市場です。");
        };
    }

    public static boolean isValidMarket(String market) {
        try {
            fromString(market);
            return true;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }
}
package simplex.bn25.zhao335952.trading.model;

import simplex.bn25.zhao335952.trading.repository.TradeRepository;

import java.io.FileWriter;
import java.io.IOException;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MarketPriceGenerator {
    private final TradeRepository tradeRepository;
    private final String marketPriceCsvFilePath;

    public MarketPriceGenerator(TradeRepository tradeRepository, String marketPriceCsvFilePath) {
        this.tradeRepository = tradeRepository;
        this.marketPriceCsvFilePath = marketPriceCsvFilePath;
    }

    /**
     * 更新市场价格 CSV 文件
     */
    public void generateMarketPriceCsv() {
        List<Trade> trades = tradeRepository.getAllTrades();
        Map<String, BigDecimal> latestPrices = new HashMap<>();

        // 遍历交易记录，找到每只股票的最新价格
        for (Trade trade : trades) {
            String ticker = trade.getTicker();
            BigDecimal tradedPrice = trade.getTradedUnitPrice();
            latestPrices.put(ticker, tradedPrice); // 使用最后一次出现的交易价格
        }

        // 输出到 CSV 文件
        try (FileWriter writer = new FileWriter(marketPriceCsvFilePath)) {
            writer.write("Ticker,Market_price\n"); // 写入标题行
            for (Map.Entry<String, BigDecimal> entry : latestPrices.entrySet()) {
                writer.write(entry.getKey() + "," + entry.getValue() + "\n");
            }
            System.out.println("实时市场价格文件已更新: " + marketPriceCsvFilePath);
        } catch (IOException e) {
            System.out.println("实时市场价格文件生成失败: " + e.getMessage());
        }
    }
}
// 新建 Side Enum
package simplex.bn25.zhao335952.trading.model;

public enum Side {
    BUY("Buy"),
    SELL("Sell");

    private final String name;

    Side(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public static Side fromString(String side) {
        if (side.equalsIgnoreCase("Buy")) {
            return BUY;
        } else if (side.equalsIgnoreCase("Sell")) {
            return SELL;
        } else {
            throw new IllegalArgumentException("Invalid side format. Use 'Buy' or 'Sell'.");
        }
    }
}
package simplex.bn25.zhao335952.trading.model;

public class Stock {
    private String ticker;
    private String productName;
    private Market market;
    private long sharesIssued;

    public Stock(String ticker, String productName, String market, long sharesIssued) {
        this.ticker = ticker;
        this.productName = productName;
        this.market = Market.valueOf(market);
        this.sharesIssued = sharesIssued;
    }

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        if (isValidTicker(ticker)) {
            this.ticker = ticker;
        } else {
            throw new IllegalArgumentException("無効な銘柄コードです。");
        }
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        if (isValidProductName(productName)) {
            this.productName = productName;
        } else {
            throw new IllegalArgumentException("無効な銘柄名です。");
        }
    }

    public String getMarket() {
        return market.getSymbol();
    }

    public void setMarket(String market) {
        this.market = Market.fromString(market);
    }

    public long getSharesIssued() {
        return sharesIssued;
    }


    public static boolean isValidTicker(String ticker) {
        return ticker.matches("^[0-9][ACDFGHJKLMNPRSTUWXYacdfghjklmnprstuwxy0-9][0-9][ACDFGHJKLMNPRSTUWXYacdfghjklmnprstuwxy0-9]$");
    }

    public static boolean isValidProductName(String productName) {
        return productName.matches("^[A-Za-z0-9 .()]+$");
    }


    public static boolean isValidSharesIssued(String input) {
        try {
            long sharesIssued = Long.parseLong(input);
            return sharesIssued >= 1 && sharesIssued <= 999999999999L;
        } catch (NumberFormatException e) {
            return false;
        }
    }

    public String getMarketFullName() {
        return market.getFullName();
    }
}
package simplex.bn25.zhao335952.trading.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class Trade {
    private final LocalDateTime tradedDatetime;
    private String ticker;
    private String tickerName;
    private String side;
    private int quantity;
    private final BigDecimal tradedUnitPrice;
    private final LocalDateTime inputDatetime;

    public static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public Trade(LocalDateTime tradedDatetime, String ticker, String tickerName, String side, int quantity, BigDecimal tradedUnitPrice, LocalDateTime inputDatetime) {
        this.tradedDatetime = tradedDatetime;
        this.ticker = ticker;
        this.tickerName = tickerName;
        setSide(side);
        setQuantity(quantity);
        this.tradedUnitPrice = tradedUnitPrice;
        this.inputDatetime = inputDatetime;
    }

    public LocalDateTime getTradedDatetime() {
        return tradedDatetime;
    }

    public String getTicker() {
        return ticker;
    }

    public void setTicker(String ticker) {
        this.ticker = ticker;
    }

    public String getTickerName() {
        return tickerName;
    }

    public void setTickerName(String tickerName) {
        this.tickerName = tickerName;
    }

    public String getSide() {
        return side;
    }

    public void setSide(String side) {
        if (isValidSide(side)) {
            this.side = side;
        } else {
            throw new IllegalArgumentException("BuyまたはSellを入力してください。");
        }
    }

    public int getQuantity() {
        return quantity;
    }

    public void setQuantity(int quantity) {
        if (quantity > 0 && quantity % 100 == 0) { // 假设每笔交易为100股的倍数
            this.quantity = quantity;
        } else {
            throw new IllegalArgumentException("100の倍数である数値を入力してください。");
        }
    }

    public BigDecimal getTradedUnitPrice() {
        return tradedUnitPrice;
    }

    private boolean isValidSide(String side) {
        return "Buy".equalsIgnoreCase(side) || "Sell".equalsIgnoreCase(side);
    }

    public String toCSVFormat() {
        return String.join(",",
                tradedDatetime.format(DATETIME_FORMATTER),
                ticker,
                tickerName,
                side,
                String.valueOf(quantity),
                tradedUnitPrice.toString(),
                inputDatetime.format(DATETIME_FORMATTER)
        );
    }
}
// 添加方法 getTickerNameByTicker
package simplex.bn25.zhao335952.trading.repository;

import simplex.bn25.zhao335952.trading.model.Market;
import simplex.bn25.zhao335952.trading.model.Stock;

import java.io.*;
import java.util.List;
import java.util.ArrayList;

public class StockRepository {
    private final String csvFilePath;

    public StockRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public List<Stock> getAllStocks() {
        List<Stock> stocks = new ArrayList<>();
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length == 4) {
                    String ticker = fields[0].trim();
                    String productName = fields[1].trim();
                    String marketString = fields[2].trim();
                    long sharesIssued = Long.parseLong(fields[3].trim());

                    Market market = Market.fromString(marketString);
                    Stock stock = new Stock(ticker, productName, market.getSymbol(), sharesIssued);
                    stocks.add(stock);
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }
        return stocks;
    }

    public boolean saveStock(Stock stock) {
        try (FileWriter writer = new FileWriter(csvFilePath, true)) {
            writer.write(String.format("\n%s,%s,%s,%d",
                    stock.getTicker(),
                    stock.getProductName(),
                    stock.getMarket(),
                    stock.getSharesIssued()));
            return true;
        } catch (IOException e) {
            System.out.println("データの書き込みにエラーが発生しました。");
            return false;
        }
    }

    public boolean isTickerRegistered(String ticker) {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields[0].trim().equalsIgnoreCase(ticker)) {
                    return true;
                }
            }
        } catch (IOException e) {
            System.out.println("データが見つかりません。新しいCSVファイルを作成します。");
        }
        return false;
    }

    public String getTickerNameByTicker(String ticker) {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length > 1 && fields[0].trim().equalsIgnoreCase(ticker)) {
                    return fields[1].trim(); // Return product name
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }
        return "Unknown";
    }
}
package simplex.bn25.zhao335952.trading.repository;

import simplex.bn25.zhao335952.trading.model.Trade;

import java.io.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

public class TradeRepository {
    private final String csvFilePath;
    private static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public TradeRepository(String csvFilePath) {
        this.csvFilePath = csvFilePath;
    }

    public List<Trade> getAllTrades() {
        List<Trade> trades = new ArrayList<>();

        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            br.readLine(); // Skip header
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length == 6) {
                    LocalDateTime tradedDatetime = LocalDateTime.parse(fields[0].trim(), DATETIME_FORMATTER);
                    String ticker = fields[1].trim();
                    String side = fields[2].trim();
                    int quantity = Integer.parseInt(fields[3].trim());
                    BigDecimal tradedUnitPrice = new BigDecimal(fields[4].trim());
                    LocalDateTime inputDatetime = LocalDateTime.parse(fields[5].trim(), DATETIME_FORMATTER);

                    Trade trade = new Trade(tradedDatetime, ticker, null, side, quantity, tradedUnitPrice, inputDatetime);
                    trades.add(trade);
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }

        return trades;
    }

    public boolean saveTrade(Trade trade) {
        try (FileWriter writer = new FileWriter(csvFilePath, true)) {
            writer.write("\n" + trade.toCSVFormat());
            return true;
        } catch (IOException e) {
            System.out.println("データの書き込み中にエラーが発生しました。");
            return false;
        }
    }

    public boolean isTickerRegistered(String ticker) {
        try (BufferedReader br = new BufferedReader(new FileReader(csvFilePath))) {
            String line;
            while ((line = br.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length > 1 && fields[1].trim().equalsIgnoreCase(ticker)) {
                    return true;
                }
            }
        } catch (IOException e) {
            System.out.println("データの読み込み中にエラーが発生しました。");
        }
        return false;
    }

}
package simplex.bn25.zhao335952.trading.view;

import java.util.Scanner;

public class MenuView {
    private final Scanner scanner;

    public MenuView() {
        scanner = new Scanner(System.in);
    }

    public String displayMainMenu() {
        System.out.println("操作するメニューを選んでください。");
        System.out.println("1. 銘柄マスタ一覧表示");
        System.out.println("2. 銘柄マスタ新規登録");
        System.out.println("3. 取引新規登録");
        System.out.println("4. 取引一覧表示");
        System.out.println("5. 保有ポジション表示");
        System.out.println("9. アプリケーションを終了する");
        System.out.print("入力してください: ");

        return scanner.nextLine().trim();
    }

    public void close() {
        scanner.close();
    }

}
package simplex.bn25.zhao335952.trading.view;

import java.util.Map;

public class PositionView {

    public void displayHoldings(Map<String, Integer> holdings, Map<String, String> tickerToNameMap) {
        if (holdings.isEmpty()) {
            System.out.println("保有データが見つかりません。");
            return;
        }

        System.out.println("-------------------------------------------------------------------------------");
        System.out.printf("| %-6s | %-30s | %-15s |%n", "Ticker", "Product Name", "Quantity");
        System.out.println("-------------------------------------------------------------------------------");

        for (Map.Entry<String, Integer> entry : holdings.entrySet()) {
            String ticker = entry.getKey();
            int quantity = entry.getValue();
            String productName = tickerToNameMap.getOrDefault(ticker, "Unknown");

            System.out.printf("| %-6s | %-30s | %15s |%n",
                    ticker.toUpperCase(),
                    truncateProductName(productName),
                    formatQuantity(quantity));
        }

        System.out.println("-------------------------------------------------------------------------------");
    }

    private String truncateProductName(String productName) {
        if (productName.length() > 30) {
            return productName.substring(0, 27) + "...";
        }
        return productName;
    }

    private String formatQuantity(int quantity) {
        return String.format("%,d", quantity);
    }
}
package simplex.bn25.zhao335952.trading.view;

import simplex.bn25.zhao335952.trading.model.Stock;
import java.util.List;

public class StockView {

    public void displayStockList(List<Stock> stocks) {
        if (stocks.isEmpty()) {
            System.out.println("データが見つかりません。CSVファイルを確認してください。");
        } else {
            System.out.println("------------------------------------------------------------------------");
            System.out.printf("| %-6s | %-30s | %-8s | %15s |%n", "Ticker", "Product Name", "Market", "Shares Issued");
            System.out.println("------------------------------------------------------------------------");
            for (Stock stock : stocks) {
                displayStock(stock);
            }
            System.out.println("------------------------------------------------------------------------");
        }
    }

    public void displayStock(Stock stock) {
        String productName = stock.getProductName();
        if (productName.length() > 30) {
            productName = productName.substring(0, 27) + "...";
        }
        System.out.printf("| %-6s | %-30s | %-8s | %15s |%n",
                stock.getTicker().toUpperCase(),
                productName,
                stock.getMarketFullName(),
                formatSharesIssued(stock.getSharesIssued()));
    }

    public void showStockAddedMessage(Stock stock) {
        System.out.println("銘柄マスタ新規登録しました:" + stock.getProductName());
    }

    private String formatSharesIssued(long sharesIssued) {
        return String.format("%,d", sharesIssued);
    }

}
package simplex.bn25.zhao335952.trading.view;

import simplex.bn25.zhao335952.trading.model.Trade;
import java.util.List;

public class TradeView {

    public void displayTradeList(List<Trade> trades) {
        if (trades.isEmpty()) {
            System.out.println("取引データが見つかりません。CSVファイルを確認してください。");
        } else {
            System.out.println("-------------------------------------------------------------------------------");
            System.out.printf("| %-19s | %-6s | %-30s | %-4s | %10s | %10s |%n",
                    "Datetime","Ticker","Product Name","Side","Quantity","Unit Price");
            System.out.println("-------------------------------------------------------------------------------");
            for (Trade trade : trades) {
                displayTrade(trade);
            }
            System.out.println("-------------------------------------------------------------------------------");
        }
    }

    public void displayTrade(Trade trade) {
        System.out.printf("| %-19s | %-6s | %-30s | %-4s | %10d | %10.2f |%n",
                trade.getTradedDatetime().format(Trade.DATETIME_FORMATTER),
                trade.getTicker().toUpperCase(),
                trade.getTickerName(),
                trade.getSide(),
                trade.getQuantity(),
                trade.getTradedUnitPrice());
    }

    public void showTradeAddedMessage(Trade trade) {
        System.out.println("取引データを新規登録しました。" + trade.getTickerName() + " (" + trade.getTicker() + ")");
    }
}
package simplex.bn25.zhao335952.trading;

import simplex.bn25.zhao335952.trading.controller.MainController;
import simplex.bn25.zhao335952.trading.controller.StockController;
import simplex.bn25.zhao335952.trading.controller.TradeController;
import simplex.bn25.zhao335952.trading.repository.StockRepository;
import simplex.bn25.zhao335952.trading.repository.TradeRepository;
import simplex.bn25.zhao335952.trading.model.MarketPriceGenerator;
import simplex.bn25.zhao335952.trading.view.MenuView;
import simplex.bn25.zhao335952.trading.view.PositionView;
import simplex.bn25.zhao335952.trading.view.StockView;
import simplex.bn25.zhao335952.trading.view.TradeView;

public class Main {
    public static void main() {
        MenuView menuView = new MenuView();
        StockView stockView = new StockView();
        TradeView tradeView = new TradeView();
        PositionView positionView = new PositionView();

        String stockCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Stock.csv";
        String tradeCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Trade.csv";
        String marketPriceCsvFilePath = "C:/Users/qqq/IdeaProjects/untitled/New code/src/csvファイル/Market_price.csv";

        StockRepository stockRepository = new StockRepository(stockCsvFilePath);
        TradeRepository tradeRepository = new TradeRepository(tradeCsvFilePath);

        MarketPriceGenerator marketPriceGenerator = new MarketPriceGenerator(tradeRepository, marketPriceCsvFilePath);
        marketPriceGenerator.generateMarketPriceCsv();

        StockController stockController = new StockController(stockRepository, stockView);
        TradeController tradeController = new TradeController(tradeRepository, stockRepository, tradeView, positionView);

        MainController mainController = new MainController(menuView, stockController, tradeController);
        mainController.start();
    }
}



csvファイル様式

Market_price.csv
Ticker,Market_price
2702,10
4478,10

Stock.csv
 7203,TOYOTA MOTOR CORPORATION,P,15794987460
 8306,Mitsubishi UFJ Financial Group Inc.,P,12337710920
 6861,Keyence Corp.,P,243207684
 6758,Sony Group Corporation,P,1248619589
 4716,Oracle Corporation Japan,S,128293071
 2702,McDonald's Holdings Co.(Japan) Ltd.,S,132960000
 8572,ACOM CO. LTD.,S,1566614098
 4816,TOEI ANIMATION CO. LTD.,S,210000000
 141a,Trial Holdings Inc.,G,122318300
 4478,Freee K.K.,G,58465623
 5842,Integral Corporation,G,34975000
 7157,LIFENET INSURANCE COMPANY,G,8027982

Trade.csv
Datetime,Ticker,Side,Quantity,Unit Price
2024-12-11 12:00,4478,Buy,50000,10,2024-12-11 22:35
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Sell,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Sell,10000,10,2024-12-11 22:44
2024-12-11 12:00,4478,Buy,50000,10,2024-12-11 22:35
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,2702,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,2702,Buy,10000,10,2024-12-11 22:44
2024-12-11 13:00,4478,Sell,10000,10,2024-12-11 22:44
2024-12-11 13:00,2702,Sell,10000,10,2024-12-11 22:44
