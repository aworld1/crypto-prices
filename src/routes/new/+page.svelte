<script>
    import { onMount, tick } from 'svelte';
    import { fade, fly, scale } from 'svelte/transition';
    import { writable } from 'svelte/store';
    
    // Import Lucide icons
    import { 
      Moon, Sun, RefreshCw, Settings, TrendingUp, TrendingDown, 
      DollarSign, Percent, BarChart2, AlertTriangle, Calendar,
      Info, Star, StarOff, HelpCircle, ExternalLink
    } from 'lucide-svelte';
  
    // Theme store
    const isDarkMode = writable(false);
    
    // Coins data with expanded information
    const coins = [
      {
        id: 'bitcoin',
        name: 'Bitcoin',
        ticker: 'BTC',
        logo: 'img/bitcoin.png',
        gradientColors: {
          light: ['#F7931A', '#FEB443', '#FFC970'],
          dark: ['#8B5500', '#B36F00', '#D98700']
        },
        historicalPrice: 102000,
        description: 'The original cryptocurrency created by Satoshi Nakamoto in 2009.',
        marketCap: 1988000000000,
        volume24h: 25600000000,
        circulatingSupply: 19500000,
        isFavorite: true
      },
      {
        id: 'ethereum',
        name: 'Ethereum',
        ticker: 'ETH',
        logo: 'img/ethereum.png',
        gradientColors: {
          light: ['#3C3C3D', '#888888', '#BBBBBB'],
          dark: ['#1A1A1A', '#444444', '#777777']
        },
        historicalPrice: 3300,
        description: 'A decentralized software platform that enables smart contracts and DApps.',
        marketCap: 396000000000,
        volume24h: 15400000000,
        circulatingSupply: 120000000,
        isFavorite: true
      },
      {
        id: 'solana',
        name: 'Solana',
        ticker: 'SOL',
        logo: 'img/solana.png',
        gradientColors: {
          light: ['#9945FF', '#DC1FFF', '#00FFA3'],
          dark: ['#4A22B0', '#9012B9', '#008F61']
        },
        historicalPrice: 245,
        description: 'High-performance blockchain supporting builder applications and marketplaces.',
        marketCap: 96500000000,
        volume24h: 3600000000,
        circulatingSupply: 394000000,
        isFavorite: false
      },
      {
        id: 'virtual-protocol',
        name: 'Virtuals Protocol',
        ticker: 'VIRTUAL',
        logo: 'img/virtuals.png',
        gradientColors: {
          light: ['#C7F1F2', '#DDF8E2', '#F0FFD6'],
          dark: ['#507F80', '#70876F', '#7A8A61']
        },
        historicalPrice: 2.7,
        description: 'An innovative DeFi protocol focused on virtual asset management.',
        marketCap: 540000000,
        volume24h: 89000000,
        circulatingSupply: 200000000,
        isFavorite: false
      }
    ];
  
    // Mock historical data (would be fetched from an API in a real app)
    const historicalDataPoints = 30;
    const historicalData = {};
    
    coins.forEach(coin => {
      const data = [];
      let price = coin.historicalPrice * 0.85; // Start at 85% of current price for trending
      
      for (let i = 0; i < historicalDataPoints; i++) {
        // Generate some random variation
        const change = (Math.random() - 0.4) * 0.05;
        price = price * (1 + change);
        
        data.push({
          date: new Date(2025, 0, 21 + i).toISOString().split('T')[0],
          price
        });
      }
      
      historicalData[coin.id] = data;
    });
  
    // App state
    let cryptoData = [];
    let isLoading = true;
    let error = null;
    let initialInvestment = 1000;
    let timeRange = 'since_start';
    let sortBy = 'value';
    let sortDirection = 'desc';
    let selectedTimeframe = 'all';
    let showOnlyFavorites = false;
    let selectedCoin = null;
    let refreshing = false;
    let lastUpdated = null;
    
    // Toggle dark mode
    function toggleDarkMode() {
      $isDarkMode = !$isDarkMode;
      document.documentElement.classList.toggle('dark', $isDarkMode);
    }
    
    // Toggle favorite status
    function toggleFavorite(id) {
      cryptoData = cryptoData.map(coin => {
        if (coin.id === id) {
          return { ...coin, isFavorite: !coin.isFavorite };
        }
        return coin;
      });
    }
    
    // Format large numbers
    function formatNumber(num, digits = 2) {
      if (num === null || num === undefined) return 'N/A';
      
      if (num >= 1000000000) {
        return `$${(num / 1000000000).toFixed(digits)}B`;
      } else if (num >= 1000000) {
        return `$${(num / 1000000).toFixed(digits)}M`;
      } else if (num >= 1000) {
        return `$${(num / 1000).toFixed(digits)}K`;
      } else {
        return `$${num.toFixed(digits)}`;
      }
    }
    
    // Format percentages
    function formatPercent(num, digits = 2) {
      if (num === null || num === undefined) return 'N/A';
      return `${(num * 100).toFixed(digits)}%`;
    }
    
    // Determine color class based on performance
    function getPerformanceClass(change) {
      if (change > 0) return 'text-green-500';
      if (change < 0) return 'text-red-500';
      return 'text-gray-500';
    }
  
    // Filter and sort data
    function processData() {
      let filtered = [...cryptoData];
      
      // Filter by favorites if enabled
      if (showOnlyFavorites) {
        filtered = filtered.filter(coin => coin.isFavorite);
      }
      
      // Sort data
      filtered.sort((a, b) => {
        let valueA, valueB;
        
        switch (sortBy) {
          case 'name':
            valueA = a.name;
            valueB = b.name;
            break;
          case 'price':
            valueA = a.currentPrice || 0;
            valueB = b.currentPrice || 0;
            break;
          case 'change':
            valueA = a.percentageChange || 0;
            valueB = b.percentageChange || 0;
            break;
          case 'value':
          default:
            valueA = a.currentValue || 0;
            valueB = b.currentValue || 0;
        }
        
        if (sortBy === 'name') {
          return sortDirection === 'asc' ? 
            valueA.localeCompare(valueB) : 
            valueB.localeCompare(valueA);
        } else {
          return sortDirection === 'asc' ? valueA - valueB : valueB - valueA;
        }
      });
      
      return filtered;
    }
    
    // Get CoinGecko URL
    function getCoinGeckoUrl(coinId) {
      return `https://www.coingecko.com/en/coins/${coinId}`;
    }
    
    // Show coin details modal
    function showCoinDetails(coin) {
      selectedCoin = coin;
    }
    
    // Close coin details modal
    function closeCoinDetails() {
      selectedCoin = null;
    }
    
    // Refresh data
    async function refreshData() {
      if (refreshing) return;
      
      refreshing = true;
      
      try {
        const realCoinIds = coins
          .map((coin) => coin.id)
          .join(',');
  
        const url = `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids=${realCoinIds}`;
        const res = await fetch(url);
        const data = await res.json();
  
        const priceMap = {};
        data.forEach((item) => {
          priceMap[item.id] = item.current_price;
        });
  
        cryptoData = coins.map((coin) => {
          const currentPrice = priceMap[coin.id] || coin.historicalPrice;
          const percentageChange = coin.historicalPrice
            ? currentPrice / coin.historicalPrice - 1
            : 0;
  
          const currentValue = initialInvestment * (1 + percentageChange);
  
          return {
            ...coin,
            currentPrice,
            percentageChange,
            currentValue,
            priceHistory: historicalData[coin.id] || []
          };
        });
        
        lastUpdated = new Date();
      } catch (error) {
        console.error('Error fetching current prices:', error);
        
        // Fallback to historical prices with some variation
        cryptoData = coins.map((coin) => {
          const randomFactor = 0.9 + Math.random() * 0.2; // ±10% variation
          const currentPrice = coin.historicalPrice * randomFactor;
          const percentageChange = randomFactor - 1;
          const currentValue = initialInvestment * randomFactor;
          
          return {
            ...coin,
            currentPrice,
            percentageChange,
            currentValue,
            priceHistory: historicalData[coin.id] || []
          };
        });
      } finally {
        isLoading = false;
        refreshing = false;
      }
    }
    
    // Calculate total portfolio value
    function getTotalPortfolioValue() {
      return cryptoData.reduce((total, coin) => total + coin.currentValue, 0);
    }
    
    // Calculate total profit/loss
    function getTotalProfitLoss() {
      const total = getTotalPortfolioValue();
      return total - (initialInvestment * cryptoData.length);
    }
    
    // Update investment amount
    function updateInvestment() {
      if (initialInvestment <= 0) {
        initialInvestment = 1000; // Reset to default if invalid
        return;
      }
      
      cryptoData = cryptoData.map(coin => {
        const currentValue = initialInvestment * (1 + coin.percentageChange);
        return { ...coin, currentValue };
      });
    }
    
    // Format date for display
    function formatDate(date) {
      return new Date(date).toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      });
    }
    
    // Initialize data on component mount
    onMount(async () => {
      // Check system preference for dark mode
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      $isDarkMode = prefersDark;
      document.documentElement.classList.toggle('dark', prefersDark);
      
      // Load data
      await refreshData();
    });
  </script>
  
  <div class="{$isDarkMode ? 'dark' : ''} transition-colors duration-300">
    <div class="min-h-screen bg-gray-50 dark:bg-gray-900 transition-colors duration-300 flex flex-col">
      <!-- Header -->
      <header class="bg-white dark:bg-gray-800 shadow-md py-4 px-6 md:px-8 transition-colors duration-300">
        <div class="max-w-7xl mx-auto flex flex-col sm:flex-row justify-between items-center">
          <div class="flex items-center mb-4 sm:mb-0">
            <h1 class="text-2xl font-bold text-indigo-600 dark:text-indigo-400 mr-4">CryptoTracker</h1>
            <span class="text-sm px-3 py-1 bg-indigo-100 dark:bg-indigo-900 text-indigo-800 dark:text-indigo-200 rounded-full">
              Live
            </span>
          </div>
          
          <div class="flex items-center space-x-4">
            <button 
              class="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-gray-700 transition-colors"
              on:click={toggleDarkMode}
              aria-label="Toggle dark mode"
            >
              {#if $isDarkMode}
                <Sun size={20} class="text-yellow-400" />
              {:else}
                <Moon size={20} class="text-gray-600" />
              {/if}
            </button>
            
            <button 
              class="flex items-center space-x-2 px-3 py-2 rounded-md bg-indigo-600 hover:bg-indigo-700 text-white transition-colors"
              on:click={refreshData}
              disabled={refreshing}
              aria-label="Refresh data"
            >
              <RefreshCw size={16} class={refreshing ? 'animate-spin' : ''} />
              <span class="hidden sm:inline">
                {refreshing ? 'Updating...' : 'Refresh Prices'}
              </span>
            </button>
          </div>
        </div>
      </header>
      
      <!-- Main content -->
      <main class="flex-grow max-w-7xl w-full mx-auto py-6 px-4 sm:px-6 lg:px-8">
        <!-- Investment controls -->
        <section class="mb-8 bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 transition-colors duration-300">
          <div class="flex flex-col lg:flex-row justify-between items-start lg:items-center space-y-4 lg:space-y-0">
            <div class="flex flex-col space-y-2">
              <h2 class="text-2xl md:text-3xl font-semibold text-gray-800 dark:text-gray-100">
                Your Crypto Portfolio
              </h2>
              <p class="text-gray-600 dark:text-gray-400">
                Tracking performance since 
                <span class="font-semibold">Jan 21, 2025</span>
              </p>
            </div>
            
            <div class="flex flex-col sm:flex-row space-y-3 sm:space-y-0 sm:space-x-4 w-full lg:w-auto">
              <div class="flex flex-col relative">
                <label for="investment" class="text-sm text-gray-600 dark:text-gray-400 mb-1">
                  Initial Investment
                </label>
                <div class="relative">
                  <span class="absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-500">
                    $
                  </span>
                  <input 
                    id="investment"
                    type="number" 
                    bind:value={initialInvestment}
                    on:blur={updateInvestment}
                    on:keypress={e => e.key === 'Enter' && updateInvestment()}
                    min="1"
                    class="pl-7 pr-4 py-2 border rounded-md w-full sm:w-36 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                  />
                </div>
              </div>
              
              <div class="flex flex-col">
                <label for="sortBy" class="text-sm text-gray-600 dark:text-gray-400 mb-1">
                  Sort By
                </label>
                <div class="flex space-x-2">
                  <select 
                    id="sortBy"
                    bind:value={sortBy}
                    class="border rounded-md py-2 px-3 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                  >
                    <option value="value">Current Value</option>
                    <option value="name">Name</option>
                    <option value="price">Price</option>
                    <option value="change">% Change</option>
                  </select>
                  
                  <button 
                    on:click={() => sortDirection = sortDirection === 'asc' ? 'desc' : 'asc'}
                    class="border rounded-md p-2 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                    aria-label={sortDirection === 'asc' ? 'Sort ascending' : 'Sort descending'}
                  >
                    {#if sortDirection === 'asc'}
                      <TrendingUp size={20} />
                    {:else}
                      <TrendingDown size={20} />
                    {/if}
                  </button>
                </div>
              </div>
              
              <div class="flex items-end">
                <button
                  on:click={() => showOnlyFavorites = !showOnlyFavorites}
                  class={`flex items-center space-x-2 px-4 py-2 rounded-md border ${
                    showOnlyFavorites 
                      ? 'bg-yellow-100 border-yellow-300 text-yellow-800 dark:bg-yellow-900 dark:border-yellow-700 dark:text-yellow-200' 
                      : 'bg-gray-100 border-gray-300 text-gray-700 dark:bg-gray-700 dark:border-gray-600 dark:text-gray-200'
                  } hover:bg-opacity-80 transition-colors`}
                  aria-label={showOnlyFavorites ? 'Show all coins' : 'Show favorites only'}
                >
                  {#if showOnlyFavorites}
                    <Star size={16} fill="currentColor" />
                    <span>Favorites</span>
                  {:else}
                    <Star size={16} />
                    <span>All Coins</span>
                  {/if}
                </button>
              </div>
            </div>
          </div>
          
          <!-- Portfolio summary -->
          {#if !isLoading && cryptoData.length > 0}
            <div class="mt-6 grid grid-cols-1 md:grid-cols-3 gap-4">
              <div class="bg-gray-100 dark:bg-gray-700 rounded-lg p-4 transition-colors duration-300">
                <div class="text-sm text-gray-500 dark:text-gray-400">Total Portfolio Value</div>
                <div class="text-2xl font-bold text-gray-900 dark:text-white">
                  ${getTotalPortfolioValue().toFixed(2)}
                </div>
              </div>
              
              <div class="bg-gray-100 dark:bg-gray-700 rounded-lg p-4 transition-colors duration-300">
                <div class="text-sm text-gray-500 dark:text-gray-400">Total Profit/Loss</div>
                <div class="text-2xl font-bold {getTotalProfitLoss() >= 0 ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'}">
                  {getTotalProfitLoss() >= 0 ? '+' : ''} ${getTotalProfitLoss().toFixed(2)}
                </div>
              </div>
              
              <div class="bg-gray-100 dark:bg-gray-700 rounded-lg p-4 transition-colors duration-300">
                <div class="text-sm text-gray-500 dark:text-gray-400">Last Updated</div>
                <div class="text-lg font-medium text-gray-900 dark:text-white">
                  {lastUpdated ? lastUpdated.toLocaleTimeString() : 'Never'}
                </div>
              </div>
            </div>
          {/if}
        </section>
        
        <!-- Crypto cards grid -->
        {#if isLoading}
          <div class="flex justify-center items-center w-full h-64">
            <div class="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-indigo-600 dark:border-indigo-400"></div>
          </div>
        {:else if error}
          <div class="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded" role="alert">
            <div class="flex items-center">
              <AlertTriangle size={24} class="mr-3" />
              <p>Error loading cryptocurrency data. Please try again later.</p>
            </div>
          </div>
        {:else}
          <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4 gap-6">
            {#each processData() as coin (coin.id)}
              <div 
                in:scale={{duration: 300, delay: 100}}
                class="bg-white dark:bg-gray-800 rounded-lg shadow-md overflow-hidden transition-transform duration-200 hover:shadow-lg transform hover:scale-[1.02] border-2 border-transparent hover:border-indigo-300 dark:hover:border-indigo-700 transition-all"
              >
                <!-- Card header with gradient -->
                <div 
                  class="h-20 relative flex items-center justify-between px-4"
                  style="background: linear-gradient(135deg, {$isDarkMode ? coin.gradientColors.dark[0] : coin.gradientColors.light[0]}, {$isDarkMode ? coin.gradientColors.dark[1] : coin.gradientColors.light[1]}, {$isDarkMode ? coin.gradientColors.dark[2] : coin.gradientColors.light[2]});"
                >
                  <div class="flex items-center space-x-3">
                    <img
                      src={coin.logo}
                      alt={`${coin.name} logo`}
                      class="w-10 h-10 object-contain"
                    />
                    <div>
                      <h2 class="text-lg font-bold text-white drop-shadow-[0_1px_2px_rgba(0,0,0,0.3)]">
                        {coin.name}
                      </h2>
                      <div class="text-sm text-white text-opacity-80">{coin.ticker}</div>
                    </div>
                  </div>
                  
                  <button 
                    on:click={(e) => {
                      e.stopPropagation();
                      toggleFavorite(coin.id);
                    }}
                    class="p-1.5 rounded-full bg-white bg-opacity-20 hover:bg-opacity-30 transition-colors"
                    aria-label={coin.isFavorite ? 'Remove from favorites' : 'Add to favorites'}
                  >
                    {#if coin.isFavorite}
                      <Star size={16} fill="white" class="text-white" />
                    {:else}
                      <Star size={16} class="text-white" />
                    {/if}
                  </button>
                </div>
                
                <!-- Card body -->
                <div class="p-4">
                  <div class="mb-4">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Current Price</div>
                    <div class="flex items-center justify-between">
                      <div class="text-xl font-bold text-gray-900 dark:text-white">
                        ${coin.currentPrice?.toFixed(2) ?? 'N/A'}
                      </div>
                      
                      <div class={`text-sm font-medium flex items-center ${getPerformanceClass(coin.percentageChange)}`}>
                        {coin.percentageChange >= 0 ? '+' : ''}
                        {(coin.percentageChange * 100).toFixed(2)}%
                        {#if coin.percentageChange > 0}
                          <TrendingUp size={16} class="ml-1" />
                        {:else if coin.percentageChange < 0}
                          <TrendingDown size={16} class="ml-1" />
                        {/if}
                      </div>
                    </div>
                  </div>
                  
                  <div class="border-t border-gray-200 dark:border-gray-700 pt-4">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Your Investment</div>
                    <div class="text-2xl font-bold text-gray-900 dark:text-white">
                      ${coin.currentValue?.toFixed(2) ?? 'N/A'}
                    </div>
                    <div class="text-sm text-gray-500 dark:text-gray-400 mt-1">
                      From initial ${initialInvestment.toFixed(2)}
                    </div>
                  </div>
                  
                  <!-- Sparkline chart placeholder -->
                  <div class="h-16 mt-3 flex items-end">
                    {#each Array(10) as _, i}
                      {@const height = 25 + Math.random() * 50}
                      <div 
                        class="flex-1 mx-px rounded-t {coin.percentageChange >= 0 ? 'bg-green-400' : 'bg-red-400'}" 
                        style="height: {height}%;"
                      ></div>
                    {/each}
                  </div>
                  
                  <!-- Action buttons -->
                  <div class="mt-4 flex space-x-2">
                    <button 
                      on:click={() => showCoinDetails(coin)}
                      class="flex-1 text-center py-2 px-4 rounded bg-indigo-100 hover:bg-indigo-200 text-indigo-700 dark:bg-indigo-900 dark:hover:bg-indigo-800 dark:text-indigo-200 transition-colors"
                    >
                      Details
                    </button>
                    
                    <a 
                      href={getCoinGeckoUrl(coin.id)} 
                      target="_blank" 
                      rel="noopener noreferrer"
                      class="py-2 px-3 rounded bg-gray-100 hover:bg-gray-200 text-gray-700 dark:bg-gray-700 dark:hover:bg-gray-600 dark:text-gray-200 transition-colors"
                    >
                      <ExternalLink size={16} />
                    </a>
                  </div>
                </div>
              </div>
            {/each}
          </div>
        {/if}
        
        <!-- Empty state if no coins match filter -->
        {#if !isLoading && processData().length === 0}
          <div class="text-center py-10">
            <div class="text-gray-400 dark:text-gray-500 mb-4">
              <AlertTriangle size={48} class="mx-auto" />
            </div>
            <h3 class="text-xl font-medium text-gray-700 dark:text-gray-300">No coins to display</h3>
            <p class="text-gray-500 dark:text-gray-400 mt-2">
              {showOnlyFavorites ? 'You have no favorite coins. Add some to your favorites!' : 'No coins match your current filters.'}
            </p>
            {#if showOnlyFavorites}
              <button
                on:click={() => showOnlyFavorites = false}
                class="mt-4 inline-flex items-center px-4 py-2 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
              >
                Show All Coins
              </button>
            {/if}
          </div>
        {/if}
      </main>
      
      <!-- Footer -->
      <footer class="bg-white dark:bg-gray-800 py-6 border-t border-gray-200 dark:border-gray-700 transition-colors duration-300">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div class="flex flex-col md:flex-row justify-between items-center">
            <div class="flex items-center mb-4 md:mb-0">
              <span class="text-gray-600 dark:text-gray-400">Data provided by</span>
              <a 
                href="https://www.coingecko.com/" 
                target="_blank" 
                rel="noopener noreferrer"
                class="ml-2 text-indigo-600 hover:text-indigo-800 dark:text-indigo-400 dark:hover:text-indigo-300 font-medium"
              >
                CoinGecko
              </a>
            </div>
            
            <div class="flex items-center space-x-6">
              <span class="text-gray-600 dark:text-gray-400">@assafica</span>
              <a
                href="https://linkedin.com/in/assafica"
                target="_blank"
                rel="noopener noreferrer"
                class="bg-[#0A66C2] text-white p-2 rounded-full hover:bg-[#004182] transition-colors"
                aria-label="Visit my LinkedIn profile"
              >
                <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 24 24">
                  <path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/>
                </svg>
              </a>
            </div>
          </div>
        </div>
      </footer>
      
      <!-- Coin details modal -->
      {#if selectedCoin}
        <div 
          class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4"
          transition:fade={{duration: 200}}
        >
          <div 
            class="bg-white dark:bg-gray-800 rounded-lg shadow-xl w-full max-w-4xl max-h-[90vh] overflow-y-auto transition-colors duration-300"
            in:fly={{y: 20, duration: 300}}
          >
            <!-- Modal header -->
            <div 
              class="p-6 border-b border-gray-200 dark:border-gray-700 flex items-center justify-between"
              style="background: linear-gradient(135deg, {$isDarkMode ? selectedCoin.gradientColors.dark[0] : selectedCoin.gradientColors.light[0]}, {$isDarkMode ? selectedCoin.gradientColors.dark[1] : selectedCoin.gradientColors.light[1]}, {$isDarkMode ? selectedCoin.gradientColors.dark[2] : selectedCoin.gradientColors.light[2]});"
            >
              <div class="flex items-center space-x-4">
                <img 
                  src={selectedCoin.logo} 
                  alt={`${selectedCoin.name} logo`}
                  class="w-12 h-12 object-contain"
                />
                <div>
                  <h3 class="text-2xl font-bold text-white">{selectedCoin.name}</h3>
                  <div class="text-white text-opacity-80">{selectedCoin.ticker}</div>
                </div>
              </div>
              
              <button 
                on:click={closeCoinDetails}
                class="text-white p-2 rounded-full hover:bg-white hover:bg-opacity-20 transition-colors"
                aria-label="Close modal"
              >
                ✕
              </button>
            </div>
            
            <!-- Modal content -->
            <div class="p-6">
              <!-- Overview section -->
              <div class="mb-8">
                <h4 class="text-lg font-semibold text-gray-800 dark:text-gray-200 mb-4">Overview</h4>
                <p class="text-gray-600 dark:text-gray-400">
                  {selectedCoin.description}
                </p>
                
                <div class="grid grid-cols-1 sm:grid-cols-3 gap-4 mt-6">
                  <div class="bg-gray-100 dark:bg-gray-700 p-4 rounded-lg">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Market Cap</div>
                    <div class="text-lg font-semibold text-gray-900 dark:text-white">
                      {formatNumber(selectedCoin.marketCap)}
                    </div>
                  </div>
                  
                  <div class="bg-gray-100 dark:bg-gray-700 p-4 rounded-lg">
                    <div class="text-sm text-gray-500 dark:text-gray-400">24h Volume</div>
                    <div class="text-lg font-semibold text-gray-900 dark:text-white">
                      {formatNumber(selectedCoin.volume24h)}
                    </div>
                  </div>
                  
                  <div class="bg-gray-100 dark:bg-gray-700 p-4 rounded-lg">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Circulating Supply</div>
                    <div class="text-lg font-semibold text-gray-900 dark:text-white">
                      {selectedCoin.circulatingSupply?.toLocaleString()} {selectedCoin.ticker}
                    </div>
                  </div>
                </div>
              </div>
              
              <!-- Price & performance -->
              <div class="mb-8">
                <h4 class="text-lg font-semibold text-gray-800 dark:text-gray-200 mb-4">Price & Performance</h4>
                
                <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
                  <div class="bg-gray-100 dark:bg-gray-700 p-4 rounded-lg">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Current Price</div>
                    <div class="text-2xl font-bold text-gray-900 dark:text-white">
                      ${selectedCoin.currentPrice?.toFixed(2)}
                    </div>
                    <div class={`text-sm font-medium mt-1 flex items-center ${getPerformanceClass(selectedCoin.percentageChange)}`}>
                      {selectedCoin.percentageChange >= 0 ? '+' : ''}
                      {(selectedCoin.percentageChange * 100).toFixed(2)}%
                      {#if selectedCoin.percentageChange > 0}
                        <TrendingUp size={14} class="ml-1" />
                      {:else if selectedCoin.percentageChange < 0}
                        <TrendingDown size={14} class="ml-1" />
                      {/if}
                    </div>
                  </div>
                  
                  <div class="bg-gray-100 dark:bg-gray-700 p-4 rounded-lg">
                    <div class="text-sm text-gray-500 dark:text-gray-400">Your Investment Value</div>
                    <div class="text-2xl font-bold text-gray-900 dark:text-white">
                      ${selectedCoin.currentValue?.toFixed(2)}
                    </div>
                    <div class="text-sm text-gray-500 dark:text-gray-400 mt-1">
                      Initial investment: ${initialInvestment}
                    </div>
                  </div>
                </div>
                
                <!-- Chart placeholder -->
                <div class="mt-6 bg-white dark:bg-gray-700 p-4 rounded-lg border border-gray-200 dark:border-gray-600">
                  <div class="flex justify-between items-center mb-4">
                    <h5 class="font-medium text-gray-800 dark:text-gray-200">Price History</h5>
                    <div class="flex">
                      <button class="px-2 py-1 text-sm bg-indigo-100 dark:bg-indigo-900 text-indigo-600 dark:text-indigo-300 rounded">
                        30D
                      </button>
                    </div>
                  </div>
                  
                  <div class="h-64 flex items-end">
                    {#each Array(30) as _, i}
                      {@const height = 20 + Math.random() * 60}
                      <div class="flex flex-col items-center flex-1">
                        <div 
                          class="w-full rounded-t {selectedCoin.percentageChange >= 0 ? 'bg-green-500' : 'bg-red-500'}" 
                          style="height: {height}%;"
                        ></div>
                        {#if i % 5 === 0}
                          <div class="text-xs text-gray-500 mt-2">
                            {i === 0 ? 'Jan 21' : i === 15 ? 'Feb 5' : 'Feb 20'}
                          </div>
                        {/if}
                      </div>
                    {/each}
                  </div>
                </div>
              </div>
              
              <!-- Action buttons -->
              <div class="flex justify-end space-x-4 mt-6">
                <button
                  on:click={closeCoinDetails}
                  class="px-4 py-2 border border-gray-300 dark:border-gray-600 text-gray-700 dark:text-gray-300 rounded-md hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors"
                >
                  Close
                </button>
                
                <a
                  href={getCoinGeckoUrl(selectedCoin.id)}
                  target="_blank" 
                  rel="noopener noreferrer"
                  class="px-4 py-2 bg-indigo-600 text-white rounded-md hover:bg-indigo-700 transition-colors flex items-center"
                >
                  <span>View on CoinGecko</span>
                  <ExternalLink size={16} class="ml-2" />
                </a>
              </div>
            </div>
          </div>
        </div>
      {/if}
    </div>
  </div>