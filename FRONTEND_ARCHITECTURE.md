# Frontend Architecture

## Stack Overview

### Core Technologies
- **Framework:** Next.js 16+ (App Router)
- **Language:** TypeScript 5.3+
- **UI Library:** React 19+
- **Styling:** Tailwind CSS 4+
- **State Management:** TanStack Query (React Query) + Zustand
- **Forms:** React Hook Form + Zod validation
- **Maps:** Mapbox GL JS or Google Maps
- **Auth:** Clerk
- **Payments:** Stripe Elements
- **Analytics:** PostHog or Mixpanel

### Why Next.js?
- Server-side rendering (SEO for cleaner profiles)
- API routes (BFF pattern for sensitive operations)
- File-based routing
- Image optimization
- Edge deployment (Vercel)

---

## Project Structure

```
apps/web/
├── src/
│   ├── app/                       # Next.js App Router
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   ├── signup/
│   │   │   └── layout.tsx
│   │   ├── (customer)/
│   │   │   ├── search/
│   │   │   ├── cleaner/[id]/
│   │   │   ├── booking/
│   │   │   │   ├── [id]/
│   │   │   │   └── checkout/
│   │   │   ├── bookings/
│   │   │   └── layout.tsx
│   │   ├── (cleaner)/
│   │   │   ├── dashboard/
│   │   │   ├── profile/
│   │   │   ├── calendar/
│   │   │   ├── earnings/
│   │   │   └── layout.tsx
│   │   ├── (admin)/
│   │   │   ├── dashboard/
│   │   │   ├── disputes/
│   │   │   ├── users/
│   │   │   └── layout.tsx
│   │   ├── api/                   # Next.js API routes (BFF)
│   │   │   ├── stripe/
│   │   │   │   └── create-intent/
│   │   │   └── auth/
│   │   ├── layout.tsx
│   │   └── page.tsx               # Landing page
│   ├── components/
│   │   ├── ui/                    # Shadcn/ui components
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── input.tsx
│   │   │   └── ...
│   │   ├── search/
│   │   │   ├── SearchFilters.tsx
│   │   │   ├── CleanerCard.tsx
│   │   │   └── SearchMap.tsx
│   │   ├── booking/
│   │   │   ├── TimeSlotPicker.tsx
│   │   │   ├── ServiceSelector.tsx
│   │   │   └── BookingSummary.tsx
│   │   ├── cleaner/
│   │   │   ├── ProfileForm.tsx
│   │   │   ├── AvailabilityCalendar.tsx
│   │   │   └── EarningsChart.tsx
│   │   └── shared/
│   │       ├── Header.tsx
│   │       ├── Footer.tsx
│   │       └── LoadingSpinner.tsx
│   ├── lib/
│   │   ├── api/                   # API client
│   │   │   ├── client.ts
│   │   │   ├── cleaners.ts
│   │   │   ├── bookings.ts
│   │   │   └── ...
│   │   ├── hooks/                 # Custom React hooks
│   │   │   ├── useAuth.ts
│   │   │   ├── useBooking.ts
│   │   │   └── useGeolocation.ts
│   │   ├── utils/
│   │   │   ├── date.ts
│   │   │   ├── currency.ts
│   │   │   └── validation.ts
│   │   ├── stores/                # Zustand stores
│   │   │   ├── bookingStore.ts
│   │   │   └── searchStore.ts
│   │   └── constants.ts
│   ├── types/
│   │   ├── api.ts
│   │   ├── models.ts
│   │   └── forms.ts
│   └── styles/
│       └── globals.css
├── public/
│   ├── images/
│   └── icons/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .env.local
├── next.config.ts
├── tailwind.config.js
├── tsconfig.json
└── package.json
```

---

## Key Features & Implementation

### 1. Authentication (Clerk)

**Why Clerk?**
- Pre-built UI components
- Social auth (Google, Apple)
- Phone verification built-in
- Role management
- Fast to integrate

**Implementation:**
```typescript
// src/app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({ children }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}

// src/middleware.ts
import { authMiddleware } from '@clerk/nextjs'

export default authMiddleware({
  publicRoutes: ['/', '/search', '/cleaner/:id'],
  ignoredRoutes: ['/api/webhooks']
})

// Protected route example
// src/app/(customer)/bookings/page.tsx
import { auth } from '@clerk/nextjs'

export default async function BookingsPage() {
  const { userId } = auth()
  if (!userId) redirect('/login')

  const bookings = await fetchBookings(userId)
  return <BookingsList bookings={bookings} />
}
```

**User Metadata (Roles):**
```typescript
// Set role on signup
await clerkClient.users.updateUserMetadata(userId, {
  publicMetadata: {
    role: 'customer' // or 'cleaner' or 'admin'
  }
})

// Custom hook
export function useUserRole() {
  const { user } = useUser()
  return user?.publicMetadata?.role as 'customer' | 'cleaner' | 'admin'
}
```

---

### 2. Cleaner Search & Discovery

**Search Page Structure:**
```typescript
// src/app/(customer)/search/page.tsx
'use client'

import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { searchCleaners } from '@/lib/api/cleaners'
import SearchFilters from '@/components/search/SearchFilters'
import CleanerCard from '@/components/search/CleanerCard'
import SearchMap from '@/components/search/SearchMap'

export default function SearchPage() {
  const [filters, setFilters] = useState({
    lat: 37.7749,
    lng: -122.4194,
    radiusMeters: 5000,
    minRating: 4.5
  })

  const { data, isLoading } = useQuery({
    queryKey: ['cleaners', filters],
    queryFn: () => searchCleaners(filters),
    staleTime: 10 * 60 * 1000  // 10 minutes
  })

  return (
    <div className="flex h-screen">
      <div className="w-1/2 overflow-y-auto p-4">
        <SearchFilters filters={filters} onChange={setFilters} />
        {isLoading ? (
          <LoadingSpinner />
        ) : (
          <div className="space-y-4">
            {data?.cleaners.map(cleaner => (
              <CleanerCard key={cleaner.id} cleaner={cleaner} />
            ))}
          </div>
        )}
      </div>
      <div className="w-1/2">
        <SearchMap cleaners={data?.cleaners || []} center={filters} />
      </div>
    </div>
  )
}
```

**CleanerCard Component:**
```typescript
// src/components/search/CleanerCard.tsx
import Image from 'next/image'
import Link from 'next/link'
import { Star, MapPin } from 'lucide-react'

interface CleanerCardProps {
  cleaner: {
    id: string
    name: string
    rating: number
    reviewCount: number
    baseRate: number
    distance: number
    profileImageUrl: string
    services: string[]
  }
}

export default function CleanerCard({ cleaner }: CleanerCardProps) {
  return (
    <Link href={`/cleaner/${cleaner.id}`}>
      <div className="border rounded-lg p-4 hover:shadow-lg transition">
        <div className="flex gap-4">
          <Image
            src={cleaner.profileImageUrl}
            alt={cleaner.name}
            width={80}
            height={80}
            className="rounded-full"
          />
          <div className="flex-1">
            <h3 className="font-semibold text-lg">{cleaner.name}</h3>
            <div className="flex items-center gap-2 text-sm text-gray-600">
              <Star className="w-4 h-4 fill-yellow-400 text-yellow-400" />
              <span>{cleaner.rating.toFixed(1)}</span>
              <span>({cleaner.reviewCount} reviews)</span>
            </div>
            <div className="flex items-center gap-1 text-sm text-gray-600">
              <MapPin className="w-4 h-4" />
              <span>{(cleaner.distance / 1000).toFixed(1)} km away</span>
            </div>
            <div className="mt-2 flex flex-wrap gap-2">
              {cleaner.services.slice(0, 3).map(service => (
                <span key={service} className="px-2 py-1 bg-blue-100 text-blue-700 rounded text-xs">
                  {service}
                </span>
              ))}
            </div>
          </div>
          <div className="text-right">
            <div className="text-2xl font-bold">${(cleaner.baseRate / 100).toFixed(0)}</div>
            <div className="text-sm text-gray-600">/hour</div>
          </div>
        </div>
      </div>
    </Link>
  )
}
```

**Map Component (Mapbox):**
```typescript
// src/components/search/SearchMap.tsx
'use client'

import { useEffect, useRef } from 'react'
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css'

mapboxgl.accessToken = process.env.NEXT_PUBLIC_MAPBOX_TOKEN!

export default function SearchMap({ cleaners, center }) {
  const mapContainer = useRef<HTMLDivElement>(null)
  const map = useRef<mapboxgl.Map | null>(null)

  useEffect(() => {
    if (!mapContainer.current) return

    map.current = new mapboxgl.Map({
      container: mapContainer.current,
      style: 'mapbox://styles/mapbox/streets-v12',
      center: [center.lng, center.lat],
      zoom: 12
    })

    // Add markers for each cleaner
    cleaners.forEach(cleaner => {
      new mapboxgl.Marker()
        .setLngLat([cleaner.lng, cleaner.lat])
        .setPopup(
          new mapboxgl.Popup().setHTML(`
            <strong>${cleaner.name}</strong><br>
            $${cleaner.baseRate / 100}/hr
          `)
        )
        .addTo(map.current!)
    })

    return () => map.current?.remove()
  }, [cleaners, center])

  return <div ref={mapContainer} className="h-full w-full" />
}
```

---

### 3. Booking Flow

**Booking Checkout Page:**
```typescript
// src/app/(customer)/booking/checkout/page.tsx
'use client'

import { useState } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { useMutation } from '@tanstack/react-query'
import { loadStripe } from '@stripe/stripe-js'
import { Elements, PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js'
import { createBookingHold, createPaymentIntent } from '@/lib/api/bookings'

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_KEY!)

export default function CheckoutPage() {
  const params = useSearchParams()
  const cleanerId = params.get('cleanerId')
  const startTime = params.get('startTime')
  const endTime = params.get('endTime')

  const [paymentIntent, setPaymentIntent] = useState(null)

  const createHoldMutation = useMutation({
    mutationFn: createBookingHold,
    onSuccess: async (hold) => {
      // Create payment intent
      const intent = await createPaymentIntent({ bookingHoldId: hold.id })
      setPaymentIntent(intent)
    }
  })

  useEffect(() => {
    createHoldMutation.mutate({ cleanerId, startTime, endTime })
  }, [])

  if (!paymentIntent) return <LoadingSpinner />

  return (
    <Elements stripe={stripePromise} options={{ clientSecret: paymentIntent.clientSecret }}>
      <CheckoutForm holdId={paymentIntent.bookingHoldId} />
    </Elements>
  )
}

function CheckoutForm({ holdId }) {
  const stripe = useStripe()
  const elements = useElements()
  const router = useRouter()

  const handleSubmit = async (e) => {
    e.preventDefault()

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/booking/success?holdId=${holdId}`
      }
    })

    if (error) {
      alert(error.message)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="max-w-md mx-auto p-6">
      <h2 className="text-2xl font-bold mb-4">Complete Payment</h2>
      <PaymentElement />
      <button
        type="submit"
        disabled={!stripe}
        className="w-full mt-4 bg-blue-600 text-white py-2 rounded"
      >
        Pay Now
      </button>
    </form>
  )
}
```

**Time Slot Picker:**
```typescript
// src/components/booking/TimeSlotPicker.tsx
'use client'

import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { getAvailableSlots } from '@/lib/api/availability'

export default function TimeSlotPicker({ cleanerId, onSelect }) {
  const [selectedDate, setSelectedDate] = useState(new Date())

  const { data: slots } = useQuery({
    queryKey: ['availability', cleanerId, selectedDate],
    queryFn: () => getAvailableSlots(cleanerId, selectedDate)
  })

  return (
    <div>
      <input
        type="date"
        value={selectedDate.toISOString().split('T')[0]}
        onChange={(e) => setSelectedDate(new Date(e.target.value))}
      />
      <div className="grid grid-cols-3 gap-2 mt-4">
        {slots?.map(slot => (
          <button
            key={slot.start}
            onClick={() => onSelect(slot)}
            className="border p-2 rounded hover:bg-blue-100"
          >
            {slot.start} - {slot.end}
          </button>
        ))}
      </div>
    </div>
  )
}
```

---

### 4. Cleaner Dashboard

**Dashboard Layout:**
```typescript
// src/app/(cleaner)/dashboard/page.tsx
import { auth } from '@clerk/nextjs'
import { redirect } from 'next/navigation'
import { getCleanerStats, getUpcomingBookings } from '@/lib/api/cleaners'

export default async function CleanerDashboard() {
  const { userId } = auth()
  if (!userId) redirect('/login')

  const [stats, upcomingBookings] = await Promise.all([
    getCleanerStats(userId),
    getUpcomingBookings(userId)
  ])

  return (
    <div className="p-6">
      <h1 className="text-3xl font-bold mb-6">Dashboard</h1>

      {/* Stats Cards */}
      <div className="grid grid-cols-4 gap-4 mb-8">
        <StatCard title="This Month" value={`$${stats.monthlyEarnings}`} />
        <StatCard title="Upcoming" value={stats.upcomingBookings} />
        <StatCard title="Rating" value={stats.rating.toFixed(1)} />
        <StatCard title="Total Jobs" value={stats.totalJobs} />
      </div>

      {/* Upcoming Bookings */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">Upcoming Bookings</h2>
        <div className="space-y-4">
          {upcomingBookings.map(booking => (
            <BookingCard key={booking.id} booking={booking} />
          ))}
        </div>
      </section>
    </div>
  )
}

function StatCard({ title, value }) {
  return (
    <div className="border rounded-lg p-4">
      <div className="text-gray-600 text-sm">{title}</div>
      <div className="text-3xl font-bold">{value}</div>
    </div>
  )
}
```

**Profile Editor:**
```typescript
// src/app/(cleaner)/profile/page.tsx
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { useMutation, useQuery } from '@tanstack/react-query'
import { getCleanerProfile, updateCleanerProfile } from '@/lib/api/cleaners'

const profileSchema = z.object({
  bio: z.string().min(50).max(500),
  baseRate: z.number().min(2000).max(20000),  // $20-$200/hr in cents
  minHours: z.number().min(1).max(8),
  radiusMeters: z.number().min(1000).max(50000),
  services: z.array(z.string())
})

type ProfileForm = z.infer<typeof profileSchema>

export default function ProfilePage() {
  const { data: profile } = useQuery({
    queryKey: ['cleaner-profile'],
    queryFn: getCleanerProfile
  })

  const { register, handleSubmit, formState: { errors } } = useForm<ProfileForm>({
    resolver: zodResolver(profileSchema),
    defaultValues: profile
  })

  const updateMutation = useMutation({
    mutationFn: updateCleanerProfile,
    onSuccess: () => alert('Profile updated!')
  })

  return (
    <form onSubmit={handleSubmit(data => updateMutation.mutate(data))} className="max-w-2xl p-6">
      <h1 className="text-3xl font-bold mb-6">Edit Profile</h1>

      <div className="mb-4">
        <label className="block mb-2">Bio</label>
        <textarea
          {...register('bio')}
          rows={4}
          className="w-full border rounded p-2"
        />
        {errors.bio && <p className="text-red-600 text-sm">{errors.bio.message}</p>}
      </div>

      <div className="mb-4">
        <label className="block mb-2">Hourly Rate ($)</label>
        <input
          type="number"
          {...register('baseRate', { valueAsNumber: true })}
          step="1"
          className="w-full border rounded p-2"
        />
        {errors.baseRate && <p className="text-red-600 text-sm">{errors.baseRate.message}</p>}
      </div>

      <button type="submit" className="bg-blue-600 text-white px-6 py-2 rounded">
        Save Changes
      </button>
    </form>
  )
}
```

**Availability Calendar:**
```typescript
// src/components/cleaner/AvailabilityCalendar.tsx
'use client'

import { useState } from 'react'
import { useMutation } from '@tanstack/react-query'
import { updateAvailability } from '@/lib/api/availability'

const DAYS = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']

export default function AvailabilityCalendar() {
  const [schedule, setSchedule] = useState({
    monday: [{ start: '09:00', end: '17:00' }],
    tuesday: [{ start: '09:00', end: '17:00' }],
    // ... etc
  })

  const updateMutation = useMutation({
    mutationFn: updateAvailability,
    onSuccess: () => alert('Availability updated!')
  })

  const handleAddSlot = (day: string) => {
    setSchedule(prev => ({
      ...prev,
      [day]: [...prev[day], { start: '09:00', end: '17:00' }]
    }))
  }

  const handleRemoveSlot = (day: string, index: number) => {
    setSchedule(prev => ({
      ...prev,
      [day]: prev[day].filter((_, i) => i !== index)
    }))
  }

  return (
    <div className="space-y-4">
      {DAYS.map(day => {
        const dayKey = day.toLowerCase()
        return (
          <div key={day} className="border rounded p-4">
            <h3 className="font-semibold mb-2">{day}</h3>
            {schedule[dayKey]?.map((slot, index) => (
              <div key={index} className="flex gap-2 items-center mb-2">
                <input
                  type="time"
                  value={slot.start}
                  onChange={(e) => {
                    const newSchedule = { ...schedule }
                    newSchedule[dayKey][index].start = e.target.value
                    setSchedule(newSchedule)
                  }}
                  className="border rounded p-1"
                />
                <span>to</span>
                <input
                  type="time"
                  value={slot.end}
                  onChange={(e) => {
                    const newSchedule = { ...schedule }
                    newSchedule[dayKey][index].end = e.target.value
                    setSchedule(newSchedule)
                  }}
                  className="border rounded p-1"
                />
                <button
                  onClick={() => handleRemoveSlot(dayKey, index)}
                  className="text-red-600"
                >
                  Remove
                </button>
              </div>
            ))}
            <button
              onClick={() => handleAddSlot(dayKey)}
              className="text-blue-600 text-sm"
            >
              + Add time slot
            </button>
          </div>
        )
      })}

      <button
        onClick={() => updateMutation.mutate(schedule)}
        className="bg-blue-600 text-white px-6 py-2 rounded"
      >
        Save Availability
      </button>
    </div>
  )
}
```

---

### 5. State Management

**TanStack Query for Server State:**
```typescript
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 minutes
      cacheTime: 10 * 60 * 1000,     // 10 minutes
      refetchOnWindowFocus: false,
      retry: 1
    }
  }
})

// Usage in component:
const { data, isLoading, error } = useQuery({
  queryKey: ['cleaner', cleanerId],
  queryFn: () => fetchCleaner(cleanerId)
})

const mutation = useMutation({
  mutationFn: updateProfile,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['cleaner'] })
  }
})
```

**Zustand for Client State:**
```typescript
// src/lib/stores/bookingStore.ts
import { create } from 'zustand'

interface BookingState {
  currentBooking: {
    cleanerId: string | null
    startTime: Date | null
    endTime: Date | null
    services: string[]
  }
  setBookingDetails: (details: Partial<BookingState['currentBooking']>) => void
  clearBooking: () => void
}

export const useBookingStore = create<BookingState>((set) => ({
  currentBooking: {
    cleanerId: null,
    startTime: null,
    endTime: null,
    services: []
  },
  setBookingDetails: (details) => set((state) => ({
    currentBooking: { ...state.currentBooking, ...details }
  })),
  clearBooking: () => set({
    currentBooking: { cleanerId: null, startTime: null, endTime: null, services: [] }
  })
}))

// Usage:
const { currentBooking, setBookingDetails } = useBookingStore()
```

---

### 6. API Client

**Centralized API Client:**
```typescript
// src/lib/api/client.ts
import axios from 'axios'

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000/v1',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// Request interceptor (add auth token)
apiClient.interceptors.request.use(async (config) => {
  const token = await getAuthToken()  // From Clerk
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Response interceptor (error handling)
apiClient.interceptors.response.use(
  (response) => response.data,
  (error) => {
    if (error.response?.status === 401) {
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default apiClient
```

**API Functions:**
```typescript
// src/lib/api/cleaners.ts
import apiClient from './client'

export async function searchCleaners(params: SearchParams) {
  return apiClient.get('/search/cleaners', { params })
}

export async function getCleaner(id: string) {
  return apiClient.get(`/cleaners/${id}`)
}

export async function updateCleanerProfile(data: ProfileData) {
  return apiClient.patch('/cleaner/profile', data)
}

// src/lib/api/bookings.ts
export async function createBookingHold(data: BookingHoldData) {
  return apiClient.post('/booking/holds', data)
}

export async function confirmBooking(holdId: string, paymentIntentId: string) {
  return apiClient.post('/booking/confirm', { holdId, paymentIntentId })
}

export async function getMyBookings() {
  return apiClient.get('/me/bookings')
}
```

---

### 7. Form Validation

**Zod Schemas:**
```typescript
// src/lib/validation/booking.ts
import { z } from 'zod'

export const bookingSchema = z.object({
  cleanerId: z.string().uuid(),
  startTime: z.date().min(new Date(), 'Start time must be in the future'),
  endTime: z.date(),
  services: z.array(z.string()).min(1, 'Select at least one service')
}).refine(
  (data) => data.endTime > data.startTime,
  { message: 'End time must be after start time', path: ['endTime'] }
)

// Usage with React Hook Form:
const { register, handleSubmit, formState: { errors } } = useForm({
  resolver: zodResolver(bookingSchema)
})
```

---

### 8. Responsive Design

**Tailwind Breakpoints:**
```css
/* Mobile First Approach */
.container {
  @apply px-4;                    /* Mobile */
  @apply md:px-8;                 /* Tablet */
  @apply lg:px-16;                /* Desktop */
}

/* Example responsive grid */
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {/* ... */}
</div>
```

---

### 9. Performance Optimizations

**Image Optimization:**
```typescript
import Image from 'next/image'

<Image
  src={cleaner.profileImageUrl}
  alt={cleaner.name}
  width={200}
  height={200}
  priority={false}  // Lazy load
  placeholder="blur"
  blurDataURL="/placeholder.jpg"
/>
```

**Code Splitting:**
```typescript
// Dynamic imports for heavy components
import dynamic from 'next/dynamic'

const SearchMap = dynamic(() => import('@/components/search/SearchMap'), {
  ssr: false,  // Client-side only
  loading: () => <LoadingSpinner />
})
```

**Prefetching:**
```typescript
// Prefetch cleaner profile on hover
<Link
  href={`/cleaner/${cleaner.id}`}
  prefetch={true}  // Next.js prefetches on hover
>
  {cleaner.name}
</Link>
```

---

### 10. SEO Optimization

**Metadata (Next.js 16+):**
```typescript
// src/app/(customer)/cleaner/[id]/page.tsx
import { Metadata } from 'next'

export async function generateMetadata({ params }): Promise<Metadata> {
  const cleaner = await getCleaner(params.id)

  return {
    title: `${cleaner.name} - Professional Cleaner in San Francisco`,
    description: `Book ${cleaner.name} for cleaning services. ${cleaner.rating}/5 stars from ${cleaner.reviewCount} reviews. Starting at $${cleaner.baseRate/100}/hr.`,
    openGraph: {
      images: [cleaner.profileImageUrl]
    }
  }
}
```

**Structured Data (JSON-LD):**
```typescript
export default function CleanerProfilePage({ cleaner }) {
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'LocalBusiness',
    'name': cleaner.name,
    'aggregateRating': {
      '@type': 'AggregateRating',
      'ratingValue': cleaner.rating,
      'reviewCount': cleaner.reviewCount
    },
    'priceRange': `$$${cleaner.baseRate / 100}/hr`
  }

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
      />
      {/* Page content */}
    </>
  )
}
```

---

### 11. Analytics & Tracking

**PostHog Integration:**
```typescript
// src/lib/analytics.ts
import posthog from 'posthog-js'

if (typeof window !== 'undefined') {
  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: 'https://app.posthog.com'
  })
}

export function trackEvent(event: string, properties?: Record<string, any>) {
  posthog.capture(event, properties)
}

// Usage:
trackEvent('booking_started', {
  cleanerId,
  totalAmount: calculateTotal()
})

trackEvent('search_performed', {
  filters: { location, radius, minRating }
})
```

---

### 12. Error Boundaries

```typescript
// src/components/ErrorBoundary.tsx
'use client'

import { Component, ReactNode } from 'react'
import * as Sentry from '@sentry/nextjs'

export class ErrorBoundary extends Component<
  { children: ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  componentDidCatch(error: Error) {
    Sentry.captureException(error)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="p-8 text-center">
          <h2 className="text-2xl font-bold">Something went wrong</h2>
          <button
            onClick={() => this.setState({ hasError: false })}
            className="mt-4 bg-blue-600 text-white px-4 py-2 rounded"
          >
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}
```

---

### 13. Testing

**Unit Tests (Jest + React Testing Library):**
```typescript
// src/components/search/CleanerCard.test.tsx
import { render, screen } from '@testing-library/react'
import CleanerCard from './CleanerCard'

describe('CleanerCard', () => {
  const mockCleaner = {
    id: '123',
    name: 'Jane Doe',
    rating: 4.8,
    reviewCount: 42,
    baseRate: 5500,
    distance: 2000,
    profileImageUrl: '/jane.jpg',
    services: ['deep_clean', 'standard']
  }

  it('renders cleaner information correctly', () => {
    render(<CleanerCard cleaner={mockCleaner} />)

    expect(screen.getByText('Jane Doe')).toBeInTheDocument()
    expect(screen.getByText('4.8')).toBeInTheDocument()
    expect(screen.getByText('$55')).toBeInTheDocument()
  })
})
```

**E2E Tests (Playwright):**
```typescript
// tests/e2e/booking-flow.spec.ts
import { test, expect } from '@playwright/test'

test('complete booking flow', async ({ page }) => {
  await page.goto('/search')

  // Search for cleaners
  await page.fill('[data-testid="location-input"]', 'San Francisco')
  await page.click('[data-testid="search-button"]')

  // Click on first cleaner
  await page.click('[data-testid="cleaner-card"]:first-child')

  // Select time slot
  await page.click('[data-testid="time-slot"]:first-child')

  // Checkout
  await page.click('[data-testid="book-now"]')

  // Fill payment details (test mode)
  await page.fill('[name="cardNumber"]', '4242424242424242')
  await page.fill('[name="cardExpiry"]', '12/34')
  await page.fill('[name="cardCvc"]', '123')

  await page.click('[data-testid="submit-payment"]')

  // Verify success
  await expect(page.locator('[data-testid="booking-success"]')).toBeVisible()
})
```

---

## Mobile App (Future - React Native)

If mobile apps are needed:

**Tech Stack:**
- **Framework:** React Native (Expo)
- **Navigation:** React Navigation
- **State:** Same as web (TanStack Query + Zustand)
- **UI:** React Native Paper or NativeBase
- **Maps:** react-native-maps
- **Payments:** Stripe SDK

**Shared Code:**
```
mobile/
├── src/
│   ├── screens/        # Mobile screens
│   ├── components/     # Mobile-specific components
│   └── shared/         # Shared with web (types, utils, API client)
```

---

## Deployment

**Vercel (Recommended for Next.js):**
```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel --prod
```

**Environment Variables:**
```env
# .env.local
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/v1
NEXT_PUBLIC_STRIPE_KEY=pk_live_...
NEXT_PUBLIC_MAPBOX_TOKEN=pk.eyJ...
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
CLERK_SECRET_KEY=sk_live_...
```

---

## Performance Targets

- **Initial Load:** <2s (Lighthouse score >90)
- **Time to Interactive:** <3s
- **First Contentful Paint:** <1s
- **Largest Contentful Paint:** <2.5s

---

## Accessibility

- Semantic HTML
- ARIA labels where needed
- Keyboard navigation
- Color contrast (WCAG AA)
- Screen reader testing

---

## Next Steps

See related docs:
- [API Specification](./API_SPECIFICATION.md)
- [Database Schema](./DATABASE_SCHEMA.md)
- [Infrastructure & Deployment](./INFRASTRUCTURE_DEPLOYMENT.md)
