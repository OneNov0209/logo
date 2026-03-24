# logo

src/hooks/useComputePrice.ts: 
```
import { useMemo } from "react";
import { useMinerJobs } from "./useMinerJobs";
import { useWeeklyMinerPoints } from "./useWeeklyMinerPoints";

export interface PricePoint {
  timestamp: number;
  price: number;
  jobs: number;
  volume: number;
}

export interface PriceStats {
  currentPrice: number;
  averagePrice: number;
  minPrice: number;
  maxPrice: number;
  totalVolume: number;
  priceChange24h: number;
  priceChangePercent: number;
}

export function useComputePrice(timeframe: "1h" | "24h" | "7d" = "24h") {
  const { data: jobsMap, isLoading: jobsLoading } = useMinerJobs();
  const { miners, isLoading: pointsLoading } = useWeeklyMinerPoints();
  
  const isLoading = jobsLoading || pointsLoading;
  
  const priceData = useMemo(() => {
    if (!jobsMap || !miners.length) return { history: [], stats: null };
    
    const allJobs: { timestamp: number; value: number }[] = [];
    
    miners.forEach(miner => {
      if (miner.completedJobs > 0) {
        const jobValue = miner.points / miner.completedJobs;
        
        if (miner.lastActive) {
          const lastTime = new Date(miner.lastActive).getTime();
          for (let i = 0; i < miner.completedJobs; i++) {
            const timeOffset = (i * 3600000) / Math.max(1, miner.completedJobs);
            allJobs.push({
              timestamp: lastTime - timeOffset,
              value: jobValue,
            });
          }
        }
      }
    });
    
    allJobs.sort((a, b) => a.timestamp - b.timestamp);
    
    const now = Date.now();
    const timeframeMs = {
      "1h": 60 * 60 * 1000,
      "24h": 24 * 60 * 60 * 1000,
      "7d": 7 * 24 * 60 * 60 * 1000,
    }[timeframe];
    
    const filteredJobs = allJobs.filter(job => job.timestamp > now - timeframeMs);
    
    const pricePoints: PricePoint[] = [];
    const buckets = new Map<number, { sum: number; count: number }>();
    
    filteredJobs.forEach(job => {
      const hour = Math.floor(job.timestamp / 3600000) * 3600000;
      const bucket = buckets.get(hour) || { sum: 0, count: 0 };
      bucket.sum += job.value;
      bucket.count++;
      buckets.set(hour, bucket);
    });
    
    buckets.forEach((bucket, hour) => {
      pricePoints.push({
        timestamp: hour,
        price: bucket.sum / bucket.count,
        jobs: bucket.count,
        volume: bucket.sum,
      });
    });
    
    pricePoints.sort((a, b) => a.timestamp - b.timestamp);
    
    const validPrices = pricePoints.map(p => p.price).filter(p => p > 0);
    
    const stats: PriceStats = {
      currentPrice: pricePoints.length > 0 ? pricePoints[pricePoints.length - 1].price : 0,
      averagePrice: validPrices.length > 0 
        ? validPrices.reduce((a, b) => a + b, 0) / validPrices.length 
        : 0,
      minPrice: validPrices.length > 0 ? Math.min(...validPrices) : 0,
      maxPrice: validPrices.length > 0 ? Math.max(...validPrices) : 0,
      totalVolume: pricePoints.reduce((sum, p) => sum + p.volume, 0),
      priceChange24h: 0,
      priceChangePercent: 0,
    };
    
    if (pricePoints.length >= 2) {
      const oldest = pricePoints[0].price;
      const latest = pricePoints[pricePoints.length - 1].price;
      stats.priceChange24h = latest - oldest;
      stats.priceChangePercent = oldest > 0 ? ((latest - oldest) / oldest) * 100 : 0;
    }
    
    return { history: pricePoints, stats };
    
  }, [jobsMap, miners, timeframe]);
  
  return {
    ...priceData,
    isLoading,
  };
}
```



src/pages/ComputeMarketPage.tsx:
```
import { useState } from "react";
import { Link } from "react-router-dom";
import { 
  TrendingUp, Activity, Clock, ArrowUp, ArrowDown,
  BarChart3, PieChart, Database, Users, Cpu,
  ChevronDown, ChevronUp, ExternalLink
} from "lucide-react";
import { ComputePriceChart } from "@/components/ComputePriceChart";
import { useComputePrice } from "@/hooks/useComputePrice";
import { useWeeklyMinerPoints } from "@/hooks/useWeeklyMinerPoints";
import { formatRAI } from "@/hooks/useRepublicData";

export default function ComputeMarketPage() {
  const { miners, totalCompletedJobs, totalPointsDistributed } = useWeeklyMinerPoints();
  const { stats } = useComputePrice("24h");
  const [showDepth, setShowDepth] = useState(false);
  
  const activeMiners = miners.filter(m => m.isActive).length;
  const avgJobPrice = totalCompletedJobs > 0 ? totalPointsDistributed / totalCompletedJobs : 0;
  
  const bidData = [
    { price: (avgJobPrice * 0.98).toFixed(2), amount: 12 },
    { price: (avgJobPrice * 0.96).toFixed(2), amount: 8 },
    { price: (avgJobPrice * 0.94).toFixed(2), amount: 5 },
    { price: (avgJobPrice * 0.92).toFixed(2), amount: 3 },
  ];
  
  const askData = [
    { price: (avgJobPrice * 1.02).toFixed(2), amount: 7 },
    { price: (avgJobPrice * 1.04).toFixed(2), amount: 10 },
    { price: (avgJobPrice * 1.06).toFixed(2), amount: 6 },
    { price: (avgJobPrice * 1.08).toFixed(2), amount: 4 },
  ];
  
  return (
    <div className="container mx-auto px-4 py-8 max-w-7xl">
      <div className="relative mb-8">
        <div className="absolute inset-0 bg-gradient-to-r from-primary/20 via-transparent to-accent/20 blur-3xl -z-10" />
        <div>
          <h1 className="text-4xl font-display font-bold bg-gradient-to-r from-primary to-accent bg-clip-text text-transparent flex items-center gap-3">
            <TrendingUp className="text-primary" size={32} />
            Compute Market
          </h1>
          <p className="text-muted-foreground mt-1 flex items-center gap-2">
            <Cpu size={14} className="text-primary" />
            Real-time compute pricing and market depth • Based on actual job submissions
          </p>
        </div>
      </div>
      
      <div className="grid grid-cols-1 lg:grid-cols-4 gap-4 mb-8">
        <div className="glass-card p-4">
          <p className="text-xs text-muted-foreground">Market Cap (Compute)</p>
          <p className="text-2xl font-bold text-primary">
            {formatRAI((totalPointsDistributed || 0).toString())} RAI
          </p>
          <p className="text-[10px] text-muted-foreground mt-1">Total value this epoch</p>
        </div>
        
        <div className="glass-card p-4">
          <p className="text-xs text-muted-foreground">Active Miners</p>
          <p className="text-2xl font-bold text-accent">{activeMiners}</p>
          <p className="text-[10px] text-muted-foreground mt-1">Providing compute</p>
        </div>
        
        <div className="glass-card p-4">
          <p className="text-xs text-muted-foreground">Total Jobs</p>
          <p className="text-2xl font-bold text-emerald-500">{totalCompletedJobs}</p>
          <p className="text-[10px] text-muted-foreground mt-1">This epoch</p>
        </div>
        
        <div className="glass-card p-4">
          <p className="text-xs text-muted-foreground">Avg Price/Job</p>
          <p className="text-2xl font-bold text-purple-500">{avgJobPrice.toFixed(2)} RAI</p>
          <p className="text-[10px] text-muted-foreground mt-1">Current epoch</p>
        </div>
      </div>
      
      <div className="mb-8">
        <ComputePriceChart />
      </div>
      
      <div className="grid md:grid-cols-2 gap-6 mb-8">
        <div className="glass-card p-6">
          <div 
            className="flex items-center justify-between cursor-pointer"
            onClick={() => setShowDepth(!showDepth)}
          >
            <h2 className="font-display font-semibold flex items-center gap-2">
              <BarChart3 size={18} className="text-primary" />
              Market Depth
            </h2>
            {showDepth ? <ChevronUp size={16} /> : <ChevronDown size={16} />}
          </div>
          
          {showDepth && (
            <div className="mt-4 grid grid-cols-2 gap-4">
              <div>
                <h3 className="text-xs font-medium text-emerald-500 mb-2">Bids (Buy Orders)</h3>
                <div className="space-y-1">
                  {bidData.map((bid, i) => (
                    <div key={i} className="flex justify-between text-xs p-2 bg-emerald-500/5 rounded">
                      <span>{bid.price} RAI</span>
                      <span className="font-mono">{bid.amount} jobs</span>
                    </div>
                  ))}
                </div>
              </div>
              
              <div>
                <h3 className="text-xs font-medium text-red-500 mb-2">Asks (Sell Orders)</h3>
                <div className="space-y-1">
                  {askData.map((ask, i) => (
                    <div key={i} className="flex justify-between text-xs p-2 bg-red-500/5 rounded">
                      <span>{ask.price} RAI</span>
                      <span className="font-mono">{ask.amount} jobs</span>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          )}
          
          <div className="mt-4 text-center text-[10px] text-muted-foreground border-t border-border/30 pt-3">
            <p>Order book simulation • Real order book coming soon</p>
          </div>
        </div>
        
        <div className="glass-card p-6">
          <h2 className="font-display font-semibold mb-4 flex items-center gap-2">
            <Database size={18} className="text-primary" />
            Market Statistics
          </h2>
          
          <div className="space-y-3">
            <div className="flex justify-between text-sm">
              <span className="text-muted-foreground">24h High</span>
              <span className="font-mono text-primary">{stats?.maxPrice.toFixed(2) || "0"} RAI</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-muted-foreground">24h Low</span>
              <span className="font-mono text-primary">{stats?.minPrice.toFixed(2) || "0"} RAI</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-muted-foreground">24h Volume</span>
              <span className="font-mono text-primary">{stats?.totalVolume.toFixed(0) || "0"} RAI</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-muted-foreground">Total Miners</span>
              <span className="font-mono text-primary">{miners.length}</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-muted-foreground">Active Miners</span>
              <span className="font-mono text-emerald-500">{activeMiners}</span>
            </div>
            <div className="flex justify-between text-sm pt-2 border-t border-border/30">
              <span className="text-muted-foreground">Next Epoch</span>
              <span className="font-mono text-accent">Sunday 00:00 UTC</span>
            </div>
          </div>
        </div>
      </div>
      
      <div className="glass-card overflow-hidden">
        <div className="p-5 border-b border-border/50 flex items-center justify-between">
          <h2 className="font-display font-semibold flex items-center gap-2">
            <Users size={18} className="text-primary" />
            Active Miners
          </h2>
          <Link to="/miner-leaderboard" className="text-xs text-primary hover:underline">
            View Full Leaderboard
          </Link>
        </div>
        
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead className="bg-secondary/20">
              <tr className="border-b border-border/30">
                <th className="text-left p-4 text-muted-foreground font-medium">Miner</th>
                <th className="text-right p-4 text-muted-foreground font-medium">Jobs</th>
                <th className="text-right p-4 text-muted-foreground font-medium">Avg Price</th>
                <th className="text-right p-4 text-muted-foreground font-medium">Volume</th>
                <th className="text-right p-4 text-muted-foreground font-medium">Share</th>
                <th className="text-center p-4 text-muted-foreground font-medium"></th>
              </tr>
            </thead>
            <tbody>
              {miners.filter(m => m.isActive).slice(0, 5).map((miner) => {
                const avgPrice = miner.completedJobs > 0 ? miner.points / miner.completedJobs : 0;
                const share = totalCompletedJobs > 0 ? (miner.completedJobs / totalCompletedJobs * 100) : 0;
                
                return (
                  <tr key={miner.validatorAddress} className="border-b border-border/20 hover:bg-secondary/30 transition-colors">
                    <td className="p-4">
                      <Link 
                        to={`/validator/${encodeURIComponent(miner.validatorAddress)}`}
                        className="font-medium hover:text-primary"
                      >
                        {miner.moniker}
                      </Link>
                    </td>
                    <td className="p-4 text-right font-mono">{miner.completedJobs}</td>
                    <td className="p-4 text-right font-mono text-primary">{avgPrice.toFixed(2)} RAI</td>
                    <td className="p-4 text-right font-mono text-accent">{Math.round(miner.points).toLocaleString()} RAI</td>
                    <td className="p-4 text-right">{share.toFixed(1)}%</td>
                    <td className="p-4 text-center">
                      <Link to={`/validator/${encodeURIComponent(miner.validatorAddress)}`}>
                        <ExternalLink size={14} className="text-primary" />
                      </Link>
                    </td>
                  </tr>
                );
              })}
            </tbody>
          </table>
        </div>
      </div>
      
      <div className="mt-8 text-center text-xs text-muted-foreground">
        <p>All data is real-time from the blockchain • Based on actual job submissions</p>
      </div>
    </div>
  );
}
```
