using System;
using System.Collections.Generic;
using System.Linq;
using cAlgo.API;
using cAlgo.API.Indicators;
using cAlgo.API.Internals;

namespace cAlgo.Robots
{
    [Robot(TimeZone = TimeZones.UTC, AccessRights = AccessRights.None)]
    public class MPXProByShadyFX : Robot
    {
        // Trade Execution Parameters
        [Parameter("Volume (in Lots)", DefaultValue = 0.02, MinValue = 0.01, Step = 0.05, Group = "Execution")]
        public double VolumeInLots { get; set; }

        [Parameter("Auto-Execute Trades", DefaultValue = true, Group = "Execution")]
        public bool AutoExecute { get; set; }

        [Parameter("Batch Mode", DefaultValue = false, Group = "Execution")]
        public bool UseBatchMode { get; set; }

        [Parameter("Max Positions", DefaultValue = 30, MinValue = 1, MaxValue = 30, Group = "Execution")]
        public int MaxPositions { get; set; }

        // Take Profit Parameters
        [Parameter("Take Profit (USD)", DefaultValue = 11.01, Group = "Profit Settings")]
        public double TakeProfitUSD { get; set; }

        [Parameter("Trailing TP Activation (USD)", DefaultValue = 3.99, Group = "Profit Settings")]
        public double TrailingStartUSD { get; set; }

        [Parameter("Trailing Step (USD)", DefaultValue = 0.32, Group = "Profit Settings")]
        public double TrailingStepUSD { get; set; }

        // Indicator Parameters
        [Parameter("EMA Fast Period", DefaultValue = 13, MinValue = 5, Group = "Trend Settings")]
        public int EmaFastPeriod { get; set; }

        [Parameter("EMA Slow Period", DefaultValue = 48, MinValue = 10, Group = "Trend Settings")]
        public int EmaSlowPeriod { get; set; }

        [Parameter("Trend Sensitivity", DefaultValue = 2, MinValue = 1, MaxValue = 5, Group = "Trend Settings")]
        public int TrendSensitivity { get; set; }

        [Parameter("Stochastic K Period", DefaultValue = 10, Group = "Momentum Settings")]
        public int StochasticKPeriod { get; set; }

        [Parameter("Stochastic D Period", DefaultValue = 3, Group = "Momentum Settings")]
        public int StochasticDPeriod { get; set; }

        [Parameter("Stochastic Smoothing", DefaultValue = 4, Group = "Momentum Settings")]
        public int StochasticSmoothing { get; set; }

        [Parameter("Momentum Threshold", DefaultValue = 32, Group = "Momentum Settings")]
        public int MomentumThreshold { get; set; }

        [Parameter("Use Volume Filter", DefaultValue = true, Group = "Volume Settings")]
        public bool UseVolumeFilter { get; set; }

        [Parameter("Volume Spike Multiplier", DefaultValue = 3.1, Group = "Volume Settings")]
        public double VolumeSpikeMultiplier { get; set; }

        [Parameter("Enable Session Filter", DefaultValue = false, Group = "Time Filters")]
        public bool EnableSessionFilter { get; set; }

        [Parameter("London Session", DefaultValue = true, Group = "Time Filters")]
        public bool LondonSession { get; set; }

        [Parameter("NY Session", DefaultValue = true, Group = "Time Filters")]
        public bool NewYorkSession { get; set; }

        [Parameter("Asia Session", DefaultValue = false, Group = "Time Filters")]
        public bool AsiaSession { get; set; }

        [Parameter("Resistance Lookback", DefaultValue = 89, MinValue = 10, Group = "Resistance Settings")]
        public int ResistanceLookback { get; set; }

        // Confirmation Parameters
        [Parameter("Confirmation Candles", DefaultValue = 3, MinValue = 3, Group = "Confirmation Settings")]
        public int ConfirmationCandles { get; set; }

        [Parameter("Min Supporting Candles", DefaultValue = 3, MinValue = 3, Group = "Confirmation Settings")]
        public int MinSupportingCandles { get; set; }

        // Visualization Parameters
        [Parameter("Show Buy/Sell Signals", DefaultValue = true, Group = "Visual Settings")]
        public bool ShowSignals { get; set; }

        [Parameter("Show Trend Zones", DefaultValue = true, Group = "Visual Settings")]
        public bool ShowTrendZones { get; set; }

        [Parameter("Show TP Levels", DefaultValue = true, Group = "Visual Settings")]
        public bool ShowTakeProfits { get; set; }

        [Parameter("Buy Signal Color", DefaultValue = "Lime", Group = "Visual Settings")]
        public string BuySignalColor { get; set; }

        [Parameter("Sell Signal Color", DefaultValue = "Red", Group = "Visual Settings")]
        public string SellSignalColor { get; set; }

        [Parameter("Show Stop Loss", DefaultValue = true, Group = "Risk Management")]
        public bool ShowStopLoss { get; set; }

        [Parameter("Show RR Ratio", DefaultValue = true, Group = "Risk Management")]
        public bool ShowRiskReward { get; set; }

        [Parameter("Show Resistance Zones", DefaultValue = true, Group = "Resistance Settings")]
        public bool ShowResistanceZones { get; set; }

        // Private fields
        private ExponentialMovingAverage _emaFast;
        private ExponentialMovingAverage _emaSlow;
        private StochasticOscillator _stochastic;
        private AverageTrueRange _atr;
        private MoneyFlowIndex _mfi;
        private RelativeStrengthIndex _rsi;
        
        private double _volumeInUnits;
        private double _pipSize;
        private int _digits;
        private double _currentAtr;
        private double _avgVolume;
        private double _volumeSpikeThreshold;
        
        private double _lastStrongResistance;
        private double _lastStrongSupport;
        private double _lastHH;
        private double _lastLH;
        private double _lastHL;
        private double _lastLL;
        
        private double _lastEntryPrice = 0;
        
        // Confirmation fields
        private bool _isBullishBias;
        private int _confirmationIndex;
        private int _supportingCandles;
        private bool _setupValidated;
        private Bars _m1Bars;
        
        // Session tracking
        private DateTime _sessionStartTime;
        private int _tradesThisSession;
        private double _totalProfit = 0;
        private Dictionary<long, double> _trailingHighWatermark = new Dictionary<long, double>();
        private Dictionary<long, DateTime> _trailingActivationTimes = new Dictionary<long, DateTime>();

        // Batch mode fields
        private readonly int[] _batchSequence = new int[] { 1, 2, 3, 2, 1, 2, 3 };
        private int _currentBatchIndex = 0;
        private int _tradesInCurrentBatch = 0;

        // Constants
        private const string BuySignalPrefix = "BUY NOW_";
        private const string SellSignalPrefix = "SELL NOW_";
        private const string TrendZonePrefix = "TREND_";
        private const string TakeProfitPrefix = "TP_";
        private const string StopLossPrefix = "SL_";
        private const string ResistancePrefix = "RES_";
        private const string SupportPrefix = "SUP_";
        private const string TrailingPrefix = "TRAIL_";
        private const string Label = "MPX Pro Trade";

        protected override void OnStart()
        {
            Print("MPX Pro initialized");
            
            // Initialize indicators
            _emaFast = Indicators.ExponentialMovingAverage(Bars.ClosePrices, EmaFastPeriod);
            _emaSlow = Indicators.ExponentialMovingAverage(Bars.ClosePrices, EmaSlowPeriod);
            _stochastic = Indicators.StochasticOscillator(StochasticKPeriod, StochasticDPeriod, StochasticSmoothing, MovingAverageType.Simple);
            _atr = Indicators.AverageTrueRange(14, MovingAverageType.Exponential);
            _mfi = Indicators.MoneyFlowIndex(14);
            _rsi = Indicators.RelativeStrengthIndex(Bars.ClosePrices, 14);
            
            // Initialize M1 bars for confirmation
            _m1Bars = MarketData.GetBars(TimeFrame.Minute);
            
            // Initialize market parameters
            _pipSize = Symbol.PipSize;
            _digits = Symbol.Digits;
            _volumeInUnits = Symbol.QuantityToVolumeInUnits(VolumeInLots);
            
            // Initialize session tracking
            _sessionStartTime = Server.Time.Date;
            _tradesThisSession = 0;
            
            // Initialize confirmation state
            ResetConfirmationState();
            
            CalculateVolumeThresholds();
            InitializeResistanceLevels();
            
            // Subscribe to position closed event
            Positions.Closed += OnPositionClosed;
        }

        private void OnPositionClosed(PositionClosedEventArgs args)
        {
            var position = args.Position;
            if (position.SymbolName == SymbolName && position.Label == Label)
            {
                _totalProfit += position.NetProfit;
                Print($"✅ Position Closed: {position.TradeType} #{position.Id} | +${position.NetProfit:F2} | Total: ${_totalProfit:F2}");
                _trailingHighWatermark.Remove(position.Id);
                _trailingActivationTimes.Remove(position.Id);
                Chart.RemoveObject(TrailingPrefix + position.Id);
                ResetConfirmationState();

                // Check if all positions are closed
                if (Positions.Count == 0 && AutoExecute)
                {
                    Print("All positions closed. Waiting for next signal...");
                    
                    // Reset batch tracking if not in batch mode
                    if (!UseBatchMode)
                    {
                        _currentBatchIndex = 0;
                        _tradesInCurrentBatch = 0;
                    }
                }
            }
        }

        private void ResetConfirmationState()
        {
            _confirmationIndex = 0;
            _supportingCandles = 0;
            _setupValidated = false;
            _isBullishBias = false;
        }

        protected override void OnTick()
        {
            // Check if we're in a new trading session
            CheckNewSession();
            
            // Check for TP achievement on every tick
            CheckForProfitTarget();
            
            // Only process on M1 bar close
            if (_m1Bars.ClosePrices.LastValue != _m1Bars.ClosePrices.Last(1))
            {
                if (_confirmationIndex > 0 && _confirmationIndex < ConfirmationCandles)
                {
                    ProcessConfirmationCandle();
                }
                else if (_confirmationIndex == 0)
                {
                    CalculateSignal();
                }
            }
        }

        private void ProcessConfirmationCandle()
        {
            _confirmationIndex++;
            var currentCandle = _m1Bars[_m1Bars.Count - 2]; // Get last closed candle

            // Check if candle confirms the bias
            if ((_isBullishBias && currentCandle.Close > currentCandle.Open) ||
                (!_isBullishBias && currentCandle.Close < currentCandle.Open))
            {
                _supportingCandles++;
                Chart.DrawIcon($"Confirm{_confirmationIndex}", ChartIconType.Star, 
                    currentCandle.OpenTime, 
                    _isBullishBias ? currentCandle.Low - Symbol.PipSize : currentCandle.High + Symbol.PipSize, 
                    Color.LimeGreen);
            }
            else
            {
                Chart.DrawIcon($"Reject{_confirmationIndex}", ChartIconType.Circle, 
                    currentCandle.OpenTime, 
                    _isBullishBias ? currentCandle.Low - Symbol.PipSize : currentCandle.High + Symbol.PipSize, 
                    Color.OrangeRed);
            }

            // Visualize confirmation progress
            Chart.DrawStaticText("ConfirmationStatus", 
                $"CONFIRMATION (Last 3 Bars): {_supportingCandles}/{MinSupportingCandles} ({_confirmationIndex}/{ConfirmationCandles})", 
                VerticalAlignment.Top, HorizontalAlignment.Left, 
                _supportingCandles >= MinSupportingCandles ? Color.Green : Color.Orange);

            // Final validation after all confirmation candles
            if (_confirmationIndex == ConfirmationCandles)
            {
                _setupValidated = _supportingCandles >= MinSupportingCandles;
                if (_setupValidated)
                {
                    Chart.DrawStaticText("SetupStatus", "SETUP VALIDATED", VerticalAlignment.Top, HorizontalAlignment.Right, Color.Green);
                    if (AutoExecute && Positions.Count < MaxPositions && _tradesThisSession < 3)
                    {
                        ExecuteTrade(_isBullishBias ? TradeType.Buy : TradeType.Sell);
                    }
                }
                else
                {
                    Chart.DrawStaticText("SetupStatus", "SETUP REJECTED", VerticalAlignment.Top, HorizontalAlignment.Right, Color.Red);
                    ResetConfirmationState();
                }
            }
        }

        private void CheckNewSession()
        {
            if (Server.Time.Date != _sessionStartTime.Date)
            {
                _sessionStartTime = Server.Time.Date;
                _tradesThisSession = 0;
                ResetConfirmationState();
                Print($"New trading session started at {_sessionStartTime}. Trade counter reset.");
            }
        }

        private void CheckForProfitTarget()
        {
            foreach (var position in Positions)
            {
                if (position.SymbolName != SymbolName || position.Label != Label) continue;
                
                double netProfit = position.NetProfit;

                // Standard TP logic
                if (netProfit >= TakeProfitUSD)
                {
                    var result = ClosePosition(position);
                    if (result.IsSuccessful)
                    {
                        _totalProfit += position.NetProfit;
                        Print($"✅ TP HIT: {position.TradeType} #{position.Id} | +${position.NetProfit:F2} | Total: ${_totalProfit:F2}");
                        _trailingHighWatermark.Remove(position.Id);
                        _trailingActivationTimes.Remove(position.Id);
                        Chart.RemoveObject(TrailingPrefix + position.Id);
                    }
                    continue;
                }

                // Trailing TP logic
                if (!_trailingHighWatermark.ContainsKey(position.Id))
                {
                    _trailingHighWatermark[position.Id] = netProfit;
                }

                if (netProfit > _trailingHighWatermark[position.Id])
                {
                    _trailingHighWatermark[position.Id] = netProfit;
                }

                // Check if trailing should be activated
                if (!_trailingActivationTimes.ContainsKey(position.Id) && netProfit >= TrailingStartUSD)
                {
                    _trailingActivationTimes[position.Id] = Server.Time;
                    Print($"🔔 Trailing TP Activated for position #{position.Id} at ${netProfit:F2}");
                    
                    double activationPrice = position.TradeType == TradeType.Buy ? Symbol.Bid : Symbol.Ask;
                    Chart.DrawIcon(TrailingPrefix + position.Id, ChartIconType.Star, Bars.Count - 1, 
                        activationPrice, Color.Gold);
                }

                // Check if trailing TP should trigger
                if (_trailingActivationTimes.ContainsKey(position.Id) && 
                    netProfit <= _trailingHighWatermark[position.Id] - TrailingStepUSD)
                {
                    var result = ClosePosition(position);
                    if (result.IsSuccessful)
                    {
                        _totalProfit += position.NetProfit;
                        Print($"🔁 Trailing TP HIT: {position.TradeType} #{position.Id} | +${position.NetProfit:F2} | Total: ${_totalProfit:F2}");
                        _trailingHighWatermark.Remove(position.Id);
                        _trailingActivationTimes.Remove(position.Id);
                        Chart.RemoveObject(TrailingPrefix + position.Id);
                    }
                }
            }
        }

        private void ExecuteTrade(TradeType tradeType)
        {
            if (!AutoExecute || _tradesThisSession >= 3 || !_setupValidated) return;

            var volumeInUnits = Symbol.NormalizeVolumeInUnits(_volumeInUnits, RoundingMode.Down);
            
            if (volumeInUnits <= 0)
            {
                Print("Error: Invalid volume calculated");
                return;
            }

            int positionsToOpen = 1;
            
            if (UseBatchMode && Positions.Count == 0)
            {
                // Start new batch sequence
                positionsToOpen = Math.Min(_batchSequence[_currentBatchIndex], MaxPositions - Positions.Count);
                Print($"Starting new batch sequence: {positionsToOpen} positions");
            }
            else if (UseBatchMode && _tradesInCurrentBatch > 0 && _tradesInCurrentBatch < _batchSequence[_currentBatchIndex])
            {
                // Continue current batch
                positionsToOpen = Math.Min(1, MaxPositions - Positions.Count);
            }

            for (int i = 0; i < positionsToOpen; i++)
            {
                if (Positions.Count >= MaxPositions) break;

                var result = ExecuteMarketOrder(
                    tradeType,
                    SymbolName,
                    volumeInUnits,
                    Label,
                    null,
                    null,
                    $"MPX Pro Trade #{Positions.Count + 1}");

                if (result.IsSuccessful)
                {
                    _lastEntryPrice = tradeType == TradeType.Buy ? Symbol.Ask : Symbol.Bid;
                    _tradesThisSession++;
                    _tradesInCurrentBatch++;
                    Print($"Opened {tradeType} position #{Positions.Count} at {_lastEntryPrice:F5} (Lot: {VolumeInLots}, Session trades: {_tradesThisSession}/3)");
                    
                    // If we've completed the current batch, move to next sequence
                    if (UseBatchMode && _tradesInCurrentBatch >= _batchSequence[_currentBatchIndex])
                    {
                        _currentBatchIndex = (_currentBatchIndex + 1) % _batchSequence.Length;
                        _tradesInCurrentBatch = 0;
                        Print($"Batch completed. Next batch sequence: {_batchSequence[_currentBatchIndex]} positions");
                    }
                }
                else
                {
                    Print($"Failed to execute {tradeType} trade: {result.Error}");
                }
            }
            
            ResetConfirmationState();
        }

        private void CalculateSignal()
        {
            // Don't open new trades if we have existing positions or in confirmation
            if (Positions.Count > 0 || _tradesThisSession >= 3 || _confirmationIndex > 0) return;

            int index = Bars.Count - 1;
            _currentAtr = _atr.Result.LastValue;
            CalculateVolumeThresholds();
            UpdateResistanceLevels();

            ClearPreviousSignals();

            bool isBullish = IsBullishSignal(index) && ConfirmWithResistance(true);
            bool isBearish = IsBearishSignal(index) && ConfirmWithResistance(false);

            if (ShowResistanceZones)
                DrawResistanceZones();

            if (ShowTrendZones)
                DrawTrendZone(isBullish, isBearish);

            if (ShowSignals)
            {
                if (isBullish)
                {
                    _isBullishBias = true;
                    DrawBuySignal(index);
                    ProcessConfirmationCandle();
                }
                else if (isBearish)
                {
                    _isBullishBias = false;
                    DrawSellSignal(index);
                    ProcessConfirmationCandle();
                }
            }

            if (ShowTakeProfits && (isBullish || isBearish))
                DrawTakeProfitLevels(index, isBullish);

            if (ShowStopLoss && (isBullish || isBearish))
                DrawStopLossLevel(index, isBullish);

            if (ShowRiskReward && (isBullish || isBearish))
                DrawRiskRewardRatio(index, isBearish);
        }

        private void InitializeResistanceLevels()
        {
            var highs = Bars.HighPrices.TakeLast(ResistanceLookback);
            var lows = Bars.LowPrices.TakeLast(ResistanceLookback);
            
            _lastStrongResistance = highs.Max();
            _lastStrongSupport = lows.Min();
            
            _lastHH = highs.Max();
            _lastLH = highs.Where(h => h < _lastHH).DefaultIfEmpty(highs.Min()).Max();
            _lastLL = lows.Min();
            _lastHL = lows.Where(l => l > _lastLL).DefaultIfEmpty(lows.Max()).Min();
        }

        private void CalculateVolumeThresholds()
        {
            double sumVolume = 0;
            int lookback = Math.Min(100, Bars.ClosePrices.Count);
            
            for (int i = 1; i <= lookback; i++)
            {
                sumVolume += Bars.TickVolumes.Last(i);
            }
            
            _avgVolume = sumVolume / lookback;
            _volumeSpikeThreshold = _avgVolume * VolumeSpikeMultiplier;
        }

        private void UpdateResistanceLevels()
        {
            double currentHigh = Bars.HighPrices.LastValue;
            double currentLow = Bars.LowPrices.LastValue;

            if (currentHigh > _lastStrongResistance * 0.9995)
            {
                _lastStrongResistance = currentHigh;
            }

            if (currentLow < _lastStrongSupport * 1.0005)
            {
                _lastStrongSupport = currentLow;
            }

            bool isSwingHigh = true;
            bool isSwingLow = true;
            
            for (int i = 1; i <= ResistanceLookback/2; i++)
            {
                if (Bars.HighPrices.Last(i) >= currentHigh) isSwingHigh = false;
                if (Bars.LowPrices.Last(i) <= currentLow) isSwingLow = false;
            }

            if (isSwingHigh)
            {
                if (currentHigh > _lastHH)
                {
                    _lastLH = _lastHH;
                    _lastHH = currentHigh;
                }
                else if (currentHigh > _lastLH)
                {
                    _lastLH = currentHigh;
                }
            }

            if (isSwingLow)
            {
                if (currentLow < _lastLL)
                {
                    _lastHL = _lastLL;
                    _lastLL = currentLow;
                }
                else if (currentLow < _lastHL)
                {
                    _lastHL = currentLow;
                }
            }
        }

        private void DrawResistanceZones()
        {
            DateTime start = Bars.OpenTimes.LastValue.AddDays(-5);
            DateTime end = Bars.OpenTimes.LastValue.AddDays(5);

            Chart.DrawRectangle(
                ResistancePrefix + Bars.Count, 
                start, 
                _lastStrongResistance, 
                end, 
                _lastStrongResistance + _currentAtr, 
                Color.FromArgb(30, 255, 0, 0),
                1, 
                LineStyle.Solid)
                .IsFilled = true;
            Chart.DrawText("Resistance_Label_" + Bars.Count, "RESISTANCE", Bars.Count - 1, _lastStrongResistance + (_currentAtr / 2), Color.White);

            Chart.DrawRectangle(
                SupportPrefix + Bars.Count, 
                start, 
                _lastStrongSupport - _currentAtr, 
                end, 
                _lastStrongSupport, 
                Color.FromArgb(30, 0, 255, 0),
                1, 
                LineStyle.Solid)
                .IsFilled = true;
            Chart.DrawText("Support_Label_" + Bars.Count, "SUPPORT", Bars.Count - 1, _lastStrongSupport - (_currentAtr / 2), Color.White);

            Chart.DrawHorizontalLine("HH_Level", _lastHH, Color.Green, 1, LineStyle.Solid);
            Chart.DrawText("HH_Label_" + Bars.Count, "HH", Bars.Count - 1, _lastHH + (5 * _pipSize), Color.Green);
            Chart.DrawHorizontalLine("LH_Level", _lastLH, Color.Red, 1, LineStyle.Solid);
            Chart.DrawText("LH_Label_" + Bars.Count, "LH", Bars.Count - 1, _lastLH + (5 * _pipSize), Color.Red);
            Chart.DrawHorizontalLine("HL_Level", _lastHL, Color.Blue, 1, LineStyle.Solid);
            Chart.DrawText("HL_Label_" + Bars.Count, "HL", Bars.Count - 1, _lastHL - (5 * _pipSize), Color.Blue);
            Chart.DrawHorizontalLine("LL_Level", _lastLL, Color.Orange, 1, LineStyle.Solid);
            Chart.DrawText("LL_Label_" + Bars.Count, "LL", Bars.Count - 1, _lastLL - (5 * _pipSize), Color.Orange);
        }

        private bool ConfirmWithResistance(bool isBullish)
        {
            double close = Bars.ClosePrices.LastValue;
            double atrBuffer = _currentAtr * 0.5;

            if (isBullish)
            {
                if (close >= _lastStrongResistance - atrBuffer)
                    return false;
                
                return close > _lastHL;
            }
            else
            {
                if (close <= _lastStrongSupport + atrBuffer)
                    return false;
                
                return close < _lastLH;
            }
        }

        private bool IsBullishSignal(int index)
        {
            bool trendConfirmed = IsBullishTrend(index);
            bool momentumConfirmed = IsBullishMomentum(index);
            bool volumeConfirmed = IsBullishVolume(index);
            bool rsiConfirmed = _rsi.Result.LastValue <= 25;
            bool volatilityFilter = _currentAtr > (CalculateAtrAverage(20) * 0.7);
            bool sessionFilter = IsGoodSession();
            bool divergenceConfirmed = IsBullishDivergence(index);

            int score = 0;
            if (trendConfirmed) score += 2;
            if (momentumConfirmed) score += 1;
            if (volumeConfirmed) score += 1;
            if (rsiConfirmed) score += 1;
            if (volatilityFilter) score += 1;
            if (sessionFilter) score += 1;
            if (divergenceConfirmed) score += 1;

            return score >= 6;
        }

        private bool IsBearishSignal(int index)
        {
            bool trendConfirmed = IsBearishTrend(index);
            bool momentumConfirmed = IsBearishMomentum(index);
            bool volumeConfirmed = IsBearishVolume(index);
            bool rsiConfirmed = _rsi.Result.LastValue >= 75;
            bool volatilityFilter = _currentAtr > (CalculateAtrAverage(20) * 0.7);
            bool sessionFilter = IsGoodSession();

            int score = 0;
            if (trendConfirmed) score += 2;
            if (momentumConfirmed) score += 1;
            if (volumeConfirmed) score += 1;
            if (rsiConfirmed) score += 1;
            if (volatilityFilter) score += 1;
            if (sessionFilter) score += 1;

            return score >= 5;
        }

        private double CalculateAtrAverage(int periods)
        {
            double sum = 0;
            int count = Math.Min(periods, _atr.Result.Count);
            
            for (int i = 0; i < count; i++)
            {
                sum += _atr.Result.Last(i);
            }
            
            return sum / count;
        }

        private bool IsBullishTrend(int index)
        {
            double fastEma = _emaFast.Result.LastValue;
            double slowEma = _emaSlow.Result.LastValue;

            bool priceAbove = Bars.ClosePrices.LastValue > slowEma;
            double sensitivityFactor = 0.0005 * _pipSize * TrendSensitivity;
            bool emaCross = fastEma > slowEma + sensitivityFactor;

            return emaCross && priceAbove;
        }

        private bool IsBearishTrend(int index)
        {
            double fastEma = _emaFast.Result.LastValue;
            double slowEma = _emaSlow.Result.LastValue;

            bool priceBelow = Bars.ClosePrices.LastValue < slowEma;
            double sensitivityFactor = 0.0005 * _pipSize * TrendSensitivity;
            bool emaCross = fastEma < slowEma - sensitivityFactor;

            return emaCross && priceBelow;
        }

        private bool IsBullishMomentum(int index)
        {
            double k = _stochastic.PercentK.LastValue;
            double d = _stochastic.PercentD.LastValue;

            bool kAboveD = k > d;
            bool notOverbought = k < (100 - MomentumThreshold);
            bool rising = k > _stochastic.PercentK[index - 1];

            return kAboveD && notOverbought && rising;
        }

        private bool IsBearishMomentum(int index)
        {
            double k = _stochastic.PercentK.LastValue;
            double d = _stochastic.PercentD.LastValue;

            bool kBelowD = k < d;
            bool notOversold = k > MomentumThreshold;
            bool falling = k < _stochastic.PercentK[index - 1];

            return kBelowD && notOversold && falling;
        }

        private bool IsBullishVolume(int index)
        {
            if (!UseVolumeFilter) return true;

            bool volumeSpike = Bars.TickVolumes.LastValue > _volumeSpikeThreshold;
            bool mfiBullish = _mfi.Result.LastValue > 50;

            return volumeSpike || mfiBullish;
        }

        private bool IsBearishVolume(int index)
        {
            if (!UseVolumeFilter) return true;

            bool volumeSpike = Bars.TickVolumes.LastValue > _volumeSpikeThreshold;
            bool mfiBearish = _mfi.Result.LastValue < 50;

            return volumeSpike || mfiBearish;
        }

        private bool IsGoodSession()
        {
            if (!EnableSessionFilter) return true;

            DateTime now = Server.Time;
            TimeSpan time = now.TimeOfDay;

            bool london = LondonSession && time >= new TimeSpan(8, 0, 0) && time < new TimeSpan(17, 0, 0);
            bool ny = NewYorkSession && time >= new TimeSpan(13, 0, 0) && time < new TimeSpan(22, 0, 0);
            bool asia = AsiaSession && time >= new TimeSpan(0, 0, 0) && time < new TimeSpan(9, 0, 0);

            return london || ny || asia;
        }

        private bool IsBullishDivergence(int index)
        {
            double priceChange = Bars.ClosePrices.LastValue - Bars.ClosePrices[index - 1];
            double stochChange = _stochastic.PercentK.LastValue - _stochastic.PercentK[index - 1];
            return priceChange > 0 && stochChange >= 0;
        }

        private double GetSafeYPosition(double baseY, string prefix, double offsetStep)
        {
            double y = baseY;
            int offsetCount = 0;
            bool overlap;

            do
            {
                overlap = false;
                foreach (var obj in Chart.Objects)
                {
                    if (obj.Name.StartsWith(prefix))
                    {
                        double objY = 0;
                        if (obj is ChartIcon)
                            objY = ((ChartIcon)obj).Y; // Use Y for price level of icons
                        else if (obj is ChartHorizontalLine)
                            objY = ((ChartHorizontalLine)obj).Y; // Use Y for price level of horizontal lines
                        else if (obj is ChartRectangle)
                            objY = Math.Min(((ChartRectangle)obj).Y1, ((ChartRectangle)obj).Y2); // Use Y1 or Y2 for rectangles

                        if (Math.Abs(objY - y) < offsetStep)
                        {
                            overlap = true;
                            break;
                        }
                    }
                }
                if (overlap)
                {
                    y = baseY + (offsetCount * offsetStep);
                    offsetCount++;
                }
            } while (overlap && offsetCount < 10); // Limit to 10 offsets to avoid infinite loop

            return y;
        }

        private void DrawBuySignal(int index)
        {
            double arrowPrice = Bars.LowPrices.LastValue - 15 * _pipSize;
            double entryPrice = Bars.ClosePrices.LastValue;
            string arrowName = BuySignalPrefix + "ARROW_" + Bars.Count;
            string entryName = BuySignalPrefix + "ENTRY_" + Bars.Count;
            string priceName = BuySignalPrefix + "PRICE_" + Bars.Count;

            arrowPrice = GetSafeYPosition(arrowPrice, BuySignalPrefix, 10 * _pipSize);
            entryPrice = GetSafeYPosition(entryPrice, BuySignalPrefix, 10 * _pipSize);

            Chart.DrawIcon(arrowName, ChartIconType.UpArrow, index, arrowPrice, Color.FromName(BuySignalColor));
            Chart.DrawIcon(entryName, ChartIconType.Circle, index, entryPrice, Color.FromName(BuySignalColor));

            Chart.DrawText(priceName, "BUY NOW: " + entryPrice.ToString($"F{_digits}"),
                index, entryPrice - (5 * _pipSize), Color.FromName(BuySignalColor))
                .VerticalAlignment = VerticalAlignment.Top;
        }

        private void DrawSellSignal(int index)
        {
            double arrowPrice = Bars.HighPrices.LastValue + 15 * _pipSize;
            double entryPrice = Bars.ClosePrices.LastValue;
            string arrowName = SellSignalPrefix + "ARROW_" + Bars.Count;
            string entryName = SellSignalPrefix + "ENTRY_" + Bars.Count;
            string priceName = SellSignalPrefix + "PRICE_" + Bars.Count;

            arrowPrice = GetSafeYPosition(arrowPrice, SellSignalPrefix, 10 * _pipSize);
            entryPrice = GetSafeYPosition(entryPrice, SellSignalPrefix, 10 * _pipSize);

            Chart.DrawIcon(arrowName, ChartIconType.DownArrow, index, arrowPrice, Color.FromName(SellSignalColor));
            Chart.DrawIcon(entryName, ChartIconType.Circle, index, entryPrice, Color.FromName(SellSignalColor));

            Chart.DrawText(priceName, "SELL NOW: " + entryPrice.ToString($"F{_digits}"),
                index, entryPrice + (5 * _pipSize), Color.FromName(SellSignalColor))
                .VerticalAlignment = VerticalAlignment.Bottom;
        }

        private void DrawTrendZone(bool isBullish, bool isBearish)
        {
            string name = TrendZonePrefix + Bars.Count;
            DateTime start = Bars.OpenTimes.LastValue;
            DateTime end = start.AddDays(1);

            if (isBullish)
            {
                Chart.DrawRectangle(name, start, Bars.LowPrices.Minimum(20), end, Bars.HighPrices.Maximum(20),
                    Color.FromArgb(20, 0, 255, 0), 1, LineStyle.Solid);
            }
            else if (isBearish)
            {
                Chart.DrawRectangle(name, start, Bars.LowPrices.Minimum(20), end, Bars.HighPrices.Maximum(20),
                    Color.FromArgb(20, 255, 0, 0), 1, LineStyle.Solid);
            }
        }

        private void DrawTakeProfitLevels(int index, bool isBullish)
        {
            double entryPrice = Bars.ClosePrices.LastValue;
            // Log inputs for debugging
            Print($"EntryPrice: {entryPrice}, Symbol.PipSize: {Symbol.PipSize}, Symbol.PipValue: {Symbol.PipValue}, _volumeInUnits: {_volumeInUnits}");
            // Calculate TP distance in pips based on USD target
            double tpDistanceInPips = TakeProfitUSD / (Symbol.PipValue * _volumeInUnits);
            Print($"tpDistanceInPips: {tpDistanceInPips}");
            double tpPrice = isBullish ? 
                entryPrice + (tpDistanceInPips * Symbol.PipSize) :
                entryPrice - (tpDistanceInPips * Symbol.PipSize);
            
            string name = TakeProfitPrefix + Bars.Count;
            Color color = Color.FromArgb(255, 215, 0); // Gold color for TP
            
            tpPrice = GetSafeYPosition(tpPrice, TakeProfitPrefix, 5 * _pipSize);
            Chart.DrawHorizontalLine(name, tpPrice, color, 2, LineStyle.DotsRare);
            // Updated line to show TP distance in pips
            Chart.DrawText(name + "_label", "TP: " + tpPrice.ToString($"F{_digits}"), index, tpPrice, color)
                  .VerticalAlignment = isBullish ? VerticalAlignment.Bottom : VerticalAlignment.Top;
        }

        private void DrawStopLossLevel(int index, bool isBullish)
        {
            double entryPrice = Bars.ClosePrices.LastValue;
            double slDistance = _currentAtr * 1.5;
            double slPrice = isBullish ? entryPrice - slDistance : entryPrice + slDistance;
            
            string name = StopLossPrefix + Bars.Count;
            slPrice = GetSafeYPosition(slPrice, StopLossPrefix, 5 * _pipSize);
            Chart.DrawHorizontalLine(name, slPrice, Color.Red, 2, LineStyle.DotsRare);
            Chart.DrawText(name + "_label", "SL " + slPrice.ToString($"F{_digits}"), index, slPrice, Color.Red)
                .VerticalAlignment = isBullish ? VerticalAlignment.Bottom : VerticalAlignment.Top;
        }

        private void DrawRiskRewardRatio(int index, bool isBearish)
        {
            double entryPrice = Bars.ClosePrices.LastValue;
            double slDistanceInPips = (_currentAtr / Symbol.PipSize) * 1.5; // Convert ATR to pips
            double tpDistanceInPips = TakeProfitUSD / (Symbol.PipValue * 667); // TP distance in pips
            Print($"slDistanceInPips: {slDistanceInPips}, tpDistanceInPips: {tpDistanceInPips}");
            double rrRatio = Math.Round(tpDistanceInPips / slDistanceInPips, 1);
            
            string text = $"RR: 1:{rrRatio}";
            double yPos = isBearish ? entryPrice - (_currentAtr * 0.5) : entryPrice + (_currentAtr * 0.5);
            yPos = GetSafeYPosition(yPos, "RR_", 5 * _pipSize);
            
            Chart.DrawText("RR_" + Bars.Count, text, index, yPos, Color.White)
                .VerticalAlignment = VerticalAlignment.Top;
        }

        private void ClearPreviousSignals()
        {
            foreach (var draw in Chart.Objects)
            {
                if (draw.Name.StartsWith(BuySignalPrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(SellSignalPrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(TrendZonePrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(TakeProfitPrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(StopLossPrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith("RR_")) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(ResistancePrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.StartsWith(SupportPrefix)) Chart.RemoveObject(draw.Name);
                if (draw.Name.EndsWith("_Level")) Chart.RemoveObject(draw.Name);
                if (draw.Name.EndsWith("_label")) Chart.RemoveObject(draw.Name);
            }
        }

        protected override void OnStop()
        {
            Print("MPX Pro stopped");
            Print($"Total Profit: ${_totalProfit:F2}");
        }
    }
}
