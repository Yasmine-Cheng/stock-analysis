# 水泥股投資分析

> 08170120鄭雅綿
> 08170222周采葳
> 08170123葉芯妤

Detail：寫一程式分析今年以來如何投資操作台灣的水泥股票，獲得較佳的投資報酬率。除繳交程式碼外，須再交WORD檔說明程式的分析結果，包含如何做投資操作。

# 投資操作說明

> - 水泥股共有7支，分別1101-1104及1108-1110
> - 分析期間：2021/01/01~2021/04/19

### 邏輯

找到每支股票在觀察期間中的最高及最低價，因此選擇時可先只查看最高(High)及最低價(Low)的欄位。

選擇出每支股票中的最高﹑最低價以及最高最低價的日期，並算出投資報酬率，公式：( 賣出價格-買入價格 ) / 買入價格。

最後選出投資報酬率最高的股票。

**注意**：有些股票的最高最低價不一定是先發生最低價再發生最高價，因此選擇時有兩種方法，1. 先選出該股票最低價，再找出發生最低價以後的最高價。2. 先選出該股票的最高價，再找出發生最高價之前的最低價。

### 步驟

1. 選擇出分析時間及股票
2. 找出最高及最低價以及發生最高最低價的日期
3. 計算價差及經過天數以導出投資報酬率→( 賣出價格-買入價格 ) / 買入價格
4. 投資投資報酬率最高的股票，並在正確入場時間及退場時間投資

# 程式碼

> 執行環境為Google Colab(有些安裝套件方式較為不同)

1. 安裝Ta-Lib(為Google Colab載Ta-Lib的方式)

    Ta-Lib：計算金融技術指標。本程式使用計算均線的功能。

    ```python
    !wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
    !tar -xzvf ta-lib-0.4.0-src.tar.gz
    %cd ta-lib
    !./configure --prefix=/usr
    !make
    !make install
    !pip install Ta-Lib
    ```

    若為一般執行環境(iDLE﹑Jupyter...)，可參考以下連結。

    需先下載好檔案(適合的使用者電腦的system type及python版本)至電腦中，並再使用終端機pip install。

    [https://blog.quantinsti.com/install-ta-lib-python/](https://blog.quantinsti.com/install-ta-lib-python/)

2. 設計為客製化程式

    若使用者想使用其他不同的日期及股票，只需調整目標資訊即可

    ```python
    start = datetime.datetime(2021,1,1)
    end = datetime.datetime(2021,3,30)
    stocklist = ['1101.TW','1102.TW','1103.TW','1104.TW','1108.TW','1109.TW','1110.TW']
    ```

3. 因DataReader的index為datatime type，包含日期及時間的內容，但因時間欄位皆為00:00:00，屬於不實用資訊，因此使用lambda(small function)將其符合格式

    而此段程式碼本程式一直重複使用，因此拉出建立function，使程式碼更加精簡。

    ```python
    def info(target):
        df_stock = web.DataReader(target,'yahoo',start,end)
        df_stock.index = df_stock.index.format(formatter=lambda x: x.strftime('%Y-%m-%d')) 
        return df_stock
    ```

4. 選出最高最低價，計算出投資報酬率，以list方式回傳投資股票名稱﹑最低最高價日期及報酬率(方便之後轉成DataFrame。雖沒有將DataFrame印出，但若想觀察整體DataFrame也已調成可隨時叫出)。其中回傳最低最高價日期原因為讓使用者知道進退場時機。

    ```python
    #先找出最低價，再找出最低價以後的最高價
    def startlowNhigh(stockname,df):
        lowdate = df["Low"][stockname].idxmin()
        lowprice = min(df["Low"][stockname])
        highdate = df["High"][stockname].loc[df["Low"][stockname].idxmin():].idxmax()
        highprice = max(df["High"][stockname].loc[df["Low"][stockname].idxmin():])
        roi = float((highprice-lowprice)/lowprice)
        output = list((stockname,lowdate,highdate,roi))
        return output
    #先找出最高價，再找出最高價以前的最低價
    def endlowNhigh(stockname,df):
        lowdate = df["Low"][stockname].loc[:df["Low"][stockname].idxmax()].idxmin()
        lowprice = min(df["Low"][stockname].loc[:df["Low"][stockname].idxmax()])
        highdate = df["High"][stockname].idxmax()
        highprice = max(df["High"][stockname])
        roi = float((highprice-lowprice)/lowprice)
        output = list((stockname,lowdate,highdate,roi))
        return output
    ```

5. 將list of list的資料型態轉成DataFrame

    設定data變數前已經先預存一個list的原因為轉成DataFrame時，可直接使用預存list當作column名稱

    ```python
    def df():
        data=[['stock', 'start', 'end', 'roi']]
        temp = info(stocklist)
        for idx in stocklist:  
            data.append(startlowNhigh(idx,temp))
            data.append(endlowNhigh(idx,temp))
        new_df = pd.DataFrame(data[1:], columns=data[0])
        return new_df
    ```

6. 主執行程式

    若直接印出最高投資報酬率使用者無法得知進出場時機及該股票名稱。
    因此使用for loop及if ... else方式比對出現最高投資報酬率時，回傳該投資報酬率的其他詳細資訊。

    ```python
    def main():
        data = df()
        for final in range(len(data)):
            if  data.iloc[final]['roi']==max(data['roi']):
                return data.iloc[final]
    ```

7. 繪成K線圖

    此function並非選出最佳投資報酬率的必要，但可使使用者更加了解該股票走勢。其中藍線表示10日均線，橘線表示30日均線(均線：過去一段時間市場的「平均成交價格」)

    `print(stock)`使用者可一次看到k線圖及數值形的執行結果，倘若使用者只想看到數值形結果可直接執行`main()`

    `ax = fig.add_subplot(1, 1, 1)`向圖中新增一個子圖。第 1 個引數代表圖中子圖的行數，第 2 個引數代表子圖的列數，第 3 個引數代表子圖在網格中的位置。
    可參考以下連結：
    [https://www.delftstack.com/zh-tw/howto/matplotlib/add-subplot-to-a-figure-matplotlib/](https://www.delftstack.com/zh-tw/howto/matplotlib/add-subplot-to-a-figure-matplotlib/)

    ```python
    def visualize(stock):
        print(stock)
        onlystock = stock.stock
        target = info(onlystock)
        #圖片大小
        fig = plt.figure(figsize=(24, 8))
        ax = fig.add_subplot(1, 1, 1)
        #設置X軸資訊
        ax.set_xticks(range(0, len(target.index), 10))
        ax.set_xticklabels(target.index[::10])
        #主要的繪至蠟燭圖(alpha為透明度參數)
        mpf.candlestick2_ochl(ax, target['Open'], target['Close'], target['High'],target['Low'], width=0.6, colorup='r', colordown='g', alpha=0.75)
        #10﹑30日均線
        sma_10 = talib.SMA(np.array(target['Close']), 10)
        sma_30 = talib.SMA(np.array(target['Close']), 30)
        #若x軸文字描述重疊，則旋轉90度
        # plt.xticks(rotation=90)
        #字體
        plt.rcParams['font.sans-serif']=['DejaVu Sans']
        #右上角小圖說明
        ax.plot(sma_10, label='10MA')
        ax.plot(sma_30, label='30MA')
        #圖片名稱
        plt.title(onlystock)
        ax.legend()
    ```

# 最終產出

投資股票：1108.TW

入場時間：2021/01/20

退場時間：2021/04/19

投資報酬率：0.674641

![image](https://github.com/Yasmine-Cheng/stock-analysis/blob/main/cement-stock/candlestickChart.png)

# 參考網址

[[Day25]繪製K線圖 - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10206894)
