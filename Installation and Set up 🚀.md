### 1. Run Installation commands 
run the following commands:
- nextjs: `pnpx create-next-app@latest <appname>`
- shadcn-ui: `pnpm dlx shadcn@latest init`
- supabase: `pnpm add @supabase/ssr @supabase/supabase-js`
- initialize shadcn-ui: `pnpm dlx shadcn@latest init`
### 2. Set up supabase 
- go to the [supabase website ](supabase.com) and create a new project
### 3. Set up Environment Variables
- in the root of your project folder create a .env file 
- on your project page in supabase go to `api docs > api settings`
- copy the project url and the anon key
- in your .env file create these variables and paste the values for your project  url and anon key:
	`NEXT_PUBLIC_SUPABASE_ANON_KEY=" <anon key> " NEXT_PUBLIC_SUPABASE_URL=" <project url> "`

### 4. Set up supabase Browser and Server Clients
- in the root of your project folder, create these folders `utils/supabase`

**Inside the supabase folder**
- create a client.ts file and paste this code
```
import { createBrowserClient } from "@supabase/ssr";

  

export function createClient() {

  return createBrowserClient(

    process.env.NEXT_PUBLIC_SUPABASE_URL!,

    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

  );

}
```
- create a server.ts file and paste this
```
import { createServerClient } from "@supabase/ssr";

import { cookies } from "next/headers";

  

export async function createClient() {

  const cookieStore = await cookies();

  

  return createServerClient(

    process.env.NEXT_PUBLIC_SUPABASE_URL!,

    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,

    {

      cookies: {

        getAll() {

          return cookieStore.getAll();

        },

        setAll(cookiesToSet) {

          try {

            cookiesToSet.forEach(({ name, value, options }) =>

              cookieStore.set(name, value, options)

            );

          } catch {

            // The `setAll` method was called from a Server Component.

            // This can be ignored if you have middleware refreshing

            // user sessions.

          }

        },

      },

    }

  );

}
```