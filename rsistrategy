using System;
using System.Linq;
using QuantConnect.Indicators;
using QuantConnect.Data.Consolidators;
using QuantConnect.Data.Market;
using QuantConnect.Orders;
using System.Collections.Generic;
using QuantConnect.Data;
using QuantConnect.Interfaces;
using Microsoft.VisualBasic;


namespace QuantConnect.Algorithm.Csharp
{
    /// <summary>
    /// 
    /// QC: 5-min RSI indicator
    ///
    /// Trading the ticker (SPY) only, on the 5-min timeframe referencing the RSI indicator.

    // -------CHANGES -----------
    // 1) Intraday - 12 Hourly - 5 minutes
    // 
    // </summary>
    
    public class QCRSIMovement : QCAlgorithm
    {
        private Security _spy;
        private Symbol _optionSymbol;
        private RelativeStrengthIndex fast;
        private RollingWindow<TradeBar> _window;
        private RollingWindow<IndicatorDataPoint> _rsiLag;
        private RollingWindow<OptionContract> _optionWindow;
		int _limitOrderId = 0;
        LimitOrder _limitOrder;
        
        OrderTicket currentOrder;
		OrderTicket profitTarget;
		OrderTicket stopLoss;
        

        public override void Initialize()
        {	
        	// backtest parameters
			// we have data for these dates locally
            var start = new DateTime(2020, 01, 01, 09, 30, 0);
            SetStartDate(start);
            SetEndDate(start.AddDays(30));
			SetCash(100000);
            
            // set security and option universe for tradebar
            _spy = AddEquity("SPY", Resolution.Minute);
            var option = AddOption("SPY", Resolution.Minute);
            
            // set filter on option universe
            _optionSymbol = option.Symbol;
			option.SetFilter(universe => 
			from symbol in universe
		    .WeeklysOnly().Expiration(TimeSpan.Zero, TimeSpan.FromDays(10))
			where symbol.ID.OptionRight != OptionRight.Put
			select symbol);

            // use the underlying equity as the benchmark
            SetBenchmark("SPY");
            
        	// Creates a Rolling Window indicator to hold 2 TradeBars
        	_window = new RollingWindow<TradeBar>(2);
        	
        	// Creates a Rolling Window indicator to hold 1 option contract 
        	_optionWindow = new RollingWindow<OptionContract>(1);
           
        	// create RSI indicator
        	fast = new RelativeStrengthIndex("SPY", 14, MovingAverageType.Wilders);
            
        	// Create a Rolling Window to keep two Indicator Data Points
			_rsiLag = new RollingWindow<IndicatorDataPoint>(2);
            
            // define our 5 minute trade bar consolidator. we can access the 5 minute bar
            // from the DataConsolidated events
            var minConsolidator = new TradeBarConsolidator(TimeSpan.FromMinutes(5));
       
            // attach our event handler. the event handler is a function that will be called each time we produce
            // a new consolidated piece of data.
            minConsolidator.DataConsolidated += OnFiveMinutes;

            // this call adds our daily consolidator to the manager to receive updates from the engine
            SubscriptionManager.AddConsolidator("SPY", minConsolidator);
        
            // we need to manually register these indicators for automatic updates
            RegisterIndicator("SPY", fast, minConsolidator);
           
            Schedule.On(DateRules.EveryDay(), TimeRules.BeforeMarketClose(_spy.Symbol, 4), () =>
            {
                CloseOpenPositions(_spy);
            });

            SetWarmUp(TimeSpan.FromDays(100));
        }
        
        private void CloseOpenPositions(Security security)
        {
            if (security.Invested)
            {
                Liquidate(security.Symbol);
            }
        }
        
        private void OnFiveMinutes(object sender, TradeBar e)
        {
            //Log(consolidated.Time.ToString("o") + " >> " + Symbol  + ">> LONG  >> 100 >> " + Portfolio[Symbol].Quantity);
            // if you want code to run every five minutes then you can run it inside of here
        	// Add SPY TradeBar in rolling window
        	fast.Update(new IndicatorDataPoint
            {
                Time = e.Time,
                Value = e.Value
            });
            
            _rsiLag.Add(fast.Current);
        }
        
        public void OnDataConsolidated(object sender, TradeBar e)
        {
            Log("OnDataConsolidated called");
            Log(e.ToString());
        }
        
        private DateTime previous;
        private HashSet<Symbol> _optionContracts = new HashSet<Symbol>();
        OptionChain chain;
        OptionContract contract;
        
        public void OnData(Slice data)
        {
			if (!_rsiLag.IsReady || !fast.IsReady )
        	{
        		_window.Add(data["SPY"]);
        		
        		return;
			}
       
        	var currBar = _window[0];                   // Current bar had index zero.
        	var pastBar = _window[1];                   // Past bar has index one.
        
        	Log($"Price: {pastBar.Time} -> {pastBar.Close} ... {currBar.Time} -> {currBar.Close}");
        	
        	var currRSI = _rsiLag[0];                   // Current SMA had index zero
        	var pastRSI = _rsiLag[_rsiLag.Count - 1];   // Oldest SMA has index of window count minus 1
        	
        	Log($"RSI: {pastRSI.Time}		if (!_rsiLag.IsReady || !fast.IsReady )-> {pastRSI.Value} ... {currRSI.Time} -> {currRSI.Value}");
        	
        	if (!Portfolio.Invested && currRSI >=29 && pastRSI < 29 && IsMarketOpen(_optionSymbol))
            {	
        		var startBreakOut=currRSI.Time;
 
        		if (currBar.Value>pastBar.Value )
        		{
        			if (data.OptionChains.TryGetValue(_optionSymbol, out chain))
					{
						var underlying = chain.Underlying;
						var closeChain = chain.Underlying.Price;
						
						//Console.WriteLine(string.Join(" ", close));

						var contracts = (
                    		from OptionContract in chain.OrderBy(x => x.Expiry)
                    		.ThenBy(x => x.Strike)
							// find OTM call strike closest to expiry
							where OptionContract.Right == OptionRight.Call
							where OptionContract.Strike > closeChain
							where DateTime.Compare(OptionContract.Expiry, startBreakOut) > 0
							select OptionContract
							).FirstOrDefault();
					
						//Console.WriteLine("Elements of Contracts:");
						//Console.WriteLine(string.Join(" ", contracts));

                		var consolidator = new TradeBarConsolidator(TimeSpan.FromMinutes(5));
                		consolidator.DataConsolidated += OnDataConsolidated;
                		SubscriptionManager.AddConsolidator(contracts.Symbol, consolidator);
                    
                		Log("Added new consolidator for " + contracts.Symbol.Value);
                    
                    	Decimal close = contracts.LastPrice;
                    	Decimal stopPrice = contracts.BidPrice * 0.99m;
                    	Decimal askPrice = contracts.AskPrice;
                    	
                    	if (contracts != null )
                    	{
                    		if (!_optionContracts.Contains(contracts.Symbol) )
                    		{
                        		_optionContracts.Add(contracts.Symbol);
                        		//Console.WriteLine("Elements of _optionContracts:"); 
        						foreach(var valu in _optionContracts) 
        							{ 
            							//Console.WriteLine(valu);
        							}
                        	
                        		var quantity = (int)Math.Floor(Portfolio.Cash / close);
                        		//SetHoldings(contracts.Symbol, quantity);
                        		currentOrder = Buy(contracts.Symbol, 200);
                        		// profitTarget = LimitOrder(contracts.Symbol, -1, bidPrice * (1.2m));
                        		var stopMarketTicket = LimitOrder(contracts.Symbol, -40, close * 1.20m);
                        		var stopMarketTicket2 = LimitOrder(contracts.Symbol, -40, close * 1.40m);
                        		var stopMarketTicket3 = LimitOrder(contracts.Symbol, -120, close * 1.60m);
                        		//stopLoss = StopMarketOrder(contracts.Symbol, -100, close * 0.98m);
                        		// var limitPrice = close * 1.01; // Sell equal or better than 1% > close.

                    		}
                    	}
					}
			    	else
            		{
                		Liquidate();
              
            		}
			    
					previous = data.Time;
            	}
            
            	foreach (var kpv in data.Bars)
            	{
                	Log($"---> OnData: {Time}, {kpv.Key.Value}, {kpv.Value.Close:0:00}");
            	}
        	}
    	}  
	}				
}
