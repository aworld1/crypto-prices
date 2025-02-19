<script>
    import { onMount } from 'svelte';
  
    const coins = [
      {
        id: 'bitcoin',
        name: 'Bitcoin',
        logo: 'src/lib/img/bitcoin.png',
        gradientColors: ['#F7931A', '#FEB443', '#FFC970'],
        historicalPrice: 102000
      },
      {
        id: 'ethereum',
        name: 'Ethereum',
        logo: 'src/lib/img/ethereum.png',
        gradientColors: ['#3C3C3D', '#888888', '#BBBBBB'],
        historicalPrice: 3300
      },
      {
        id: 'solana',
        name: 'Solana',
        logo: 'src/lib/img/solana.png',
        gradientColors: ['#00FFA3', '#DC1FFF', '#9945FF'],
        historicalPrice: 245
      },
      {
        id: 'virtual-protocol',
        name: 'Virtuals Protocol',
        logo: 'src/lib/img/virtuals.png',
        gradientColors: ['#C7F1F2', '#DDF8E2', '#F0FFD6'],
        historicalPrice: 2.7
      }
    ];
  
    const initialInvestment = 1000;
  
    let cryptoData = [];
    let isLoading = true;
  
    onMount(async () => {
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
          const currentPrice = priceMap[coin.id];
  
          const percentageChange = coin.historicalPrice
            ? currentPrice / coin.historicalPrice
            : 1;
  
          const currentValue = initialInvestment * percentageChange;
  
          return {
            ...coin,
            currentPrice,
            percentageChange,
            currentValue
          };
        });
      } catch (error) {
        console.error('Error fetching current prices:', error);
        cryptoData = coins.map((coin) => ({
          ...coin,
          currentPrice: coin.historicalPrice,
          percentageChange: 1,
          currentValue: initialInvestment
        }));
      } finally {
        isLoading = false;
      }
    });

    function getCoinGeckoUrl(coinId) {
        return `https://www.coingecko.com/en/coins/${coinId}`;
    }
  </script>
  
  <div class="min-h-screen bg-gray-50 flex flex-col items-center justify-center p-4">
    <h1 class="text-4xl md:text-5xl text-center mb-8 leading-tight font-semibold w-[800px] mb-16">
      If you invested 
      <b class="font-extrabold">$1000</b> 
    on the 
      <b class="font-extrabold">first day of class</b>, you would have
    </h1>
  
    {#if isLoading}
      <div class="flex justify-center items-center w-full max-w-4xl h-96">
        <div class="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-blue-500"></div>
      </div>
    {:else}
      <div class="grid grid-cols-1 sm:grid-cols-2 gap-6 w-full max-w-4xl">
        {#each cryptoData as coin (coin.id)}
          <a
            href={getCoinGeckoUrl(coin.id)}
            target="_blank"
            rel="noopener noreferrer"
            class="block transform hover:scale-[1.02] transition-transform duration-200 cursor-pointer"
          >
            <div
              class="rounded-lg shadow-md p-6 flex flex-row items-center justify-between relative h-48"
              style="background: linear-gradient(135deg, {coin.gradientColors[0]}, {coin.gradientColors[1]}, {coin.gradientColors[2]});"
            >
              <img
                src={coin.logo}
                alt={`${coin.name} logo`}
                class="w-16 h-16 object-contain ml-4 flex-shrink-0"
              />
  
              <div class="flex flex-col text-right mr-4 text-white flex-grow">
                <h2 class="text-2xl font-semibold mb-3 drop-shadow-[0_2px_4px_rgba(0,0,0,0.4)]">
                  {coin.name}
                </h2>
                <p class="text-3xl font-bold drop-shadow-[0_2px_4px_rgba(0,0,0,0.4)]">
                  ${
                    coin.currentValue 
                      ? coin.currentValue.toFixed(2) 
                      : 'Loading...'
                  }
                </p>
              </div>
            </div>
          </a>
        {/each}
      </div>
    {/if}

    <div class="fixed bottom-4 right-4 flex items-center gap-2">
      <span class="text-gray-600">@assafica</span>
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
