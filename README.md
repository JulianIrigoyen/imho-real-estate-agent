AI Real Estate Agent - Project Bootstrap Guide
Project Overview
Building an AI-powered real estate agent for the Argentine market using Vercel AI SDK, with integrations for ZonaProp, MercadoLibre, and Los Robles.
Quick Start
1. Initialize the Project
bashnpx create-next-app@latest ai-real-estate-agent --typescript --tailwind --app
cd ai-real-estate-agent
2. Install Dependencies
bash# Core AI dependencies
npm install ai @ai-sdk/openai @ai-sdk/anthropic zod

# UI components
npm install @radix-ui/react-dialog @radix-ui/react-select lucide-react
npm install @radix-ui/react-tabs @radix-ui/react-scroll-area

# Web scraping and data processing
npm install playwright cheerio p-queue p-retry

# Database (for caching property data)
npm install @vercel/postgres drizzle-orm drizzle-kit

# WhatsApp integration (optional)
npm install @whiskeysockets/baileys qrcode

# Development dependencies
npm install -D @types/node dotenv
3. Environment Setup
Create a .env.local file:
env# AI Provider Keys
OPENAI_API_KEY=your_openai_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here

# Database
POSTGRES_URL="your_vercel_postgres_url"
POSTGRES_PRISMA_URL="your_vercel_postgres_url"
POSTGRES_URL_NON_POOLING="your_vercel_postgres_url"
POSTGRES_USER="default"
POSTGRES_HOST="your-host"
POSTGRES_PASSWORD="your-password"
POSTGRES_DATABASE="verceldb"

# Optional: WhatsApp Business API
WHATSAPP_API_KEY=your_whatsapp_key
WHATSAPP_PHONE_ID=your_phone_id

# Optional: Proxy for scraping
BRIGHT_DATA_PROXY=your_proxy_url
4. Project Structure
ai-real-estate-agent/
├── app/
│   ├── api/
│   │   ├── chat/
│   │   │   └── route.ts          # Main chat endpoint
│   │   ├── search/
│   │   │   └── route.ts          # Property search API
│   │   └── scrape/
│   │       └── route.ts          # Scraping endpoints
│   ├── chat/
│   │   └── page.tsx              # Chat interface
│   ├── layout.tsx
│   └── page.tsx                  # Landing page
├── components/
│   ├── chat/
│   │   ├── chat-interface.tsx
│   │   ├── message-list.tsx
│   │   └── property-card.tsx
│   └── ui/                       # Shadcn UI components
├── lib/
│   ├── ai/
│   │   ├── agents/
│   │   │   └── real-estate-agent.ts
│   │   ├── tools/
│   │   │   ├── property-search.ts
│   │   │   ├── property-analyzer.ts
│   │   │   └── url-builder.ts
│   │   └── prompts.ts
│   ├── scrapers/
│   │   ├── zonaprop.ts
│   │   ├── mercadolibre.ts
│   │   └── los-robles.ts
│   ├── db/
│   │   ├── schema.ts
│   │   └── queries.ts
│   └── utils/
│       └── property-formatter.ts
└── types/
    └── property.ts
5. Core Implementation Files
Create these essential files to get started:
app/api/chat/route.ts
typescriptimport { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { createRealEstateAgent } from '@/lib/ai/agents/real-estate-agent';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: openai('gpt-4-turbo'),
    messages,
    tools: createRealEstateAgent(),
    system: `You are an expert real estate agent specializing in the Argentine market, 
    particularly Mar del Plata. You help users find properties, provide market insights, 
    and answer questions about neighborhoods, prices, and property features.
    
    Always be helpful, professional, and provide accurate information based on current listings.
    When showing properties, include key details like price, size, location, and amenities.`,
  });

  return result.toDataStreamResponse();
}
lib/ai/tools/property-search.ts
typescriptimport { tool } from 'ai';
import { z } from 'zod';
import { searchZonaProp } from '@/lib/scrapers/zonaprop';
import { searchMercadoLibre } from '@/lib/scrapers/mercadolibre';

export const propertySearchTool = tool({
  description: 'Search for properties across Argentine real estate platforms',
  parameters: z.object({
    location: z.string().describe('Location to search (e.g., "mar-del-plata", "palermo")'),
    propertyType: z.enum(['casa', 'departamento', 'ph']).optional(),
    minPrice: z.number().optional().describe('Minimum price in USD'),
    maxPrice: z.number().optional().describe('Maximum price in USD'),
    minSize: z.number().optional().describe('Minimum size in m²'),
    maxSize: z.number().optional().describe('Maximum size in m²'),
    bedrooms: z.number().optional(),
    platform: z.enum(['all', 'zonaprop', 'mercadolibre', 'losrobles']).default('all'),
  }),
  execute: async ({ location, propertyType, minPrice, maxPrice, minSize, maxSize, bedrooms, platform }) => {
    const results = [];

    if (platform === 'all' || platform === 'zonaprop') {
      const zonapropResults = await searchZonaProp({
        location,
        propertyType,
        minPrice,
        maxPrice,
        minSize,
        maxSize,
        bedrooms,
      });
      results.push(...zonapropResults);
    }

    if (platform === 'all' || platform === 'mercadolibre') {
      const mlResults = await searchMercadoLibre({
        location,
        propertyType,
        minPrice,
        maxPrice,
      });
      results.push(...mlResults);
    }

    return {
      properties: results,
      count: results.length,
      platforms: platform === 'all' ? ['zonaprop', 'mercadolibre'] : [platform],
    };
  },
});
lib/scrapers/zonaprop.ts
typescriptimport { chromium } from 'playwright';

interface SearchParams {
  location: string;
  propertyType?: string;
  minPrice?: number;
  maxPrice?: number;
  minSize?: number;
  maxSize?: number;
  bedrooms?: number;
}

export async function searchZonaProp(params: SearchParams) {
  const url = buildZonaPropUrl(params);
  
  const browser = await chromium.launch({ 
    headless: true,
    // Add proxy if needed
    // proxy: { server: process.env.BRIGHT_DATA_PROXY }
  });
  
  const page = await browser.newPage();
  
  try {
    await page.goto(url, { waitUntil: 'networkidle' });
    
    // Wait for listings to load
    await page.waitForSelector('.postings-container', { timeout: 10000 });
    
    const properties = await page.evaluate(() => {
      const listings = Array.from(document.querySelectorAll('.posting-card'));
      
      return listings.map(listing => ({
        id: listing.getAttribute('data-id') || '',
        title: listing.querySelector('.posting-title')?.textContent?.trim() || '',
        price: listing.querySelector('.price-tag-text-sr-only')?.textContent?.trim() || '',
        location: listing.querySelector('.posting-location')?.textContent?.trim() || '',
        size: listing.querySelector('[data-qa="POSTING_CARD_FEATURES"] span')?.textContent?.trim() || '',
        bedrooms: listing.querySelector('[data-qa="POSTING_CARD_FEATURES_BEDROOMS"]')?.textContent?.trim() || '',
        url: listing.querySelector('a')?.href || '',
        image: listing.querySelector('img')?.src || '',
        platform: 'zonaprop'
      }));
    });
    
    return properties;
  } finally {
    await browser.close();
  }
}

function buildZonaPropUrl(params: SearchParams): string {
  const { location, propertyType, minPrice, maxPrice, minSize, maxSize } = params;
  
  let url = `https://www.zonaprop.com.ar/`;
  
  // Property type
  if (propertyType) {
    url += `${propertyType}s-`;
  }
  
  url += `venta-${location}`;
  
  // Size range
  if (minSize && maxSize) {
    url += `-${minSize}-${maxSize}-m2-cubiertos`;
  } else if (minSize) {
    url += `-mas-${minSize}-m2-cubiertos`;
  } else if (maxSize) {
    url += `-menos-${maxSize}-m2-cubiertos`;
  }
  
  // Price range
  if (minPrice && maxPrice) {
    url += `-${minPrice}-${maxPrice}-dolar`;
  } else if (minPrice) {
    url += `-mas-${minPrice}-dolar`;
  } else if (maxPrice) {
    url += `-menos-${maxPrice}-dolar`;
  }
  
  url += '.html';
  
  return url;
}
components/chat/chat-interface.tsx
typescript'use client';

import { useChat } from 'ai/react';
import { useState } from 'react';
import { Send } from 'lucide-react';

export function ChatInterface() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat();
  
  return (
    <div className="flex flex-col h-[600px] w-full max-w-4xl mx-auto border rounded-lg">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((message) => (
          <div
            key={message.id}
            className={`flex ${
              message.role === 'user' ? 'justify-end' : 'justify-start'
            }`}
          >
            <div
              className={`max-w-[80%] rounded-lg p-4 ${
                message.role === 'user'
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-100 text-gray-900'
              }`}
            >
              {message.content}
            </div>
          </div>
        ))}
      </div>
      
      <form onSubmit={handleSubmit} className="border-t p-4 flex gap-2">
        <input
          type="text"
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about properties in Mar del Plata..."
          className="flex-1 px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:bg-gray-300"
        >
          <Send className="w-5 h-5" />
        </button>
      </form>
    </div>
  );
}
6. Database Schema (Optional)
If you want to cache property data:
sql-- lib/db/schema.sql
CREATE TABLE properties (
  id SERIAL PRIMARY KEY,
  external_id VARCHAR(255) UNIQUE NOT NULL,
  platform VARCHAR(50) NOT NULL,
  title TEXT NOT NULL,
  price DECIMAL(10, 2),
  currency VARCHAR(10),
  location TEXT,
  neighborhood VARCHAR(255),
  size_m2 INTEGER,
  bedrooms INTEGER,
  bathrooms INTEGER,
  property_type VARCHAR(50),
  url TEXT,
  image_url TEXT,
  description TEXT,
  amenities JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_properties_location ON properties(location);
CREATE INDEX idx_properties_price ON properties(price);
CREATE INDEX idx_properties_type ON properties(property_type);
7. Next Steps

Set up Vercel project:
bashvercel

Connect PostgreSQL database in Vercel dashboard
Add API keys to Vercel environment variables
Deploy:
bashvercel --prod


8. Testing the Agent
Start the development server:
bashnpm run dev
Visit http://localhost:3000/chat and try queries like:

"Show me apartments in Mar del Plata under $250,000"
"What are the best neighborhoods for families?"
"Find 3-bedroom houses in Güemes with a garden"

Additional Features to Implement

WhatsApp Integration using Baileys or WhatsApp Business API
Property Analytics with charts using Recharts
Saved Searches with user authentication
Email Notifications for new listings
Virtual Tours integration
Mortgage Calculator tool
Neighborhood Insights with crime stats, schools, etc.

This setup gives you a solid foundation to build upon!