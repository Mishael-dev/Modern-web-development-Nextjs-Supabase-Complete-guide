## Setup Google Auth
- go to your project page on supabase and go to authentication > providers 
- toggle google on and then copy the callback url
- go to [google cloud console](console.cloud.google.com) and create a new project
- search for and select oauth consent screen and click on external and proceed
- click next and fill out the necessary information
- on credentials > create credentials > oauth client id
- choose web application and fill in the app name
- paste the callback url you copied from firebase into the authorized redirect url's field and click on create
- copy the client id and client secret and paste it in the providers> google section in firebase
- toggle skip nounce check on and then click save

### Create user profile table
run this sql script to create user profile tables in supabase
```create table

public.profiles (

    id uuid not null references auth.users on delete cascade,

    full_name text,

    email text unique,

    avatar_url text,

    primary key (id)

);

alter table profiles enable row level security;

create policy "Public profiles are viewable by everyone." on profiles

for select

using (true);

create policy "Users can insert their own profile." on profiles

for insert

with

check (auth.uid() = id);

create policy "Users can update own profile." on profiles

for update

using (auth.uid() = id);

create function public.handle_new_user() returns trigger as $$

begin

    insert into public.profiles (id, full_name, avatar_url, email)

    values (

        new.id,

        new.raw_user_meta_data->>'full_name',

        new.raw_user_meta_data->>'avatar_url',

        new.raw_user_meta_data->>'email'

    );

    return new;

end;

$$ language plpgsql security definer;

create trigger on_auth_user_created

after insert on auth.users

for each row

execute procedure public.handle_new_user();
```

### Setting up supabase for authentication in project folder

- create a middleware.ts file in utils>supabase and paste this
```import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request,
  })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => request.cookies.set(name, value))
          supabaseResponse = NextResponse.next({
            request,
          })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // Do not run code between createServerClient and
  // supabase.auth.getUser(). A simple mistake could make it very hard to debug
  // issues with users being randomly logged out.

  // IMPORTANT: DO NOT REMOVE auth.getUser()

  const {
    data: { user },
  } = await supabase.auth.getUser()

  if (
    !user &&
    !request.nextUrl.pathname.startsWith('/login') &&
    !request.nextUrl.pathname.startsWith('/auth')
  ) {
    // no user, potentially respond by redirecting the user to the login page
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    return NextResponse.redirect(url)
  }

  // IMPORTANT: You *must* return the supabaseResponse object as it is.
  // If you're creating a new response object with NextResponse.next() make sure to:
  // 1. Pass the request in it, like so:
  //    const myNewResponse = NextResponse.next({ request })
  // 2. Copy over the cookies, like so:
  //    myNewResponse.cookies.setAll(supabaseResponse.cookies.getAll())
  // 3. Change the myNewResponse object to fit your needs, but avoid changing
  //    the cookies!
  // 4. Finally:
  //    return myNewResponse
  // If this is not done, you may be causing the browser and server to go out
  // of sync and terminate the user's session prematurely!

  return supabaseResponse
}
```

- create a middleware.ts file in the root of your project and paste this: 
```import { type NextRequest } from 'next/server'
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for the ones starting with:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * Feel free to modify this pattern to include more paths.
     */
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

- create an auth-actions.ts file in /lib and paste this;
```"use server";

  

import { revalidatePath } from "next/cache";

import { redirect } from "next/navigation";

import { createClient } from "@/utils/supabase/server";

  

export async function login(formData: FormData) {

  const supabase = await createClient();

  

  // type-casting here for convenience

  // in practice, you should validate your inputs

  const data = {

    email: formData.get("email") as string,

    password: formData.get("password") as string,

  };

  

  const { error } = await supabase.auth.signInWithPassword(data);

  

  if (error) {

    console.log(error)

    redirect("/error");

  }

  

  revalidatePath("/", "layout");

  redirect("/");

}

  

export async function signup(formData: FormData) {

  const supabase = await createClient();

  

  // type-casting here for convenience

  // in practice, you should validate your inputs

  const firstName = formData.get("first-name") as string;

  const lastName = formData.get("last-name") as string;

  const data = {

    email: formData.get("email") as string,

    password: formData.get("password") as string,

    options: {

      data: {

        full_name: `${firstName + " " + lastName}`,

        email: formData.get("email") as string,

      },

    },

  };

  

  const { error } = await supabase.auth.signUp(data);

  

  if (error) {

    console.log(error)

    redirect("/error");

  }

  

  revalidatePath("/", "layout");

  redirect("/");

}

  

export async function signout() {

  const supabase = await createClient();

  const { error } = await supabase.auth.signOut();

  if (error) {

    console.log(error);

    redirect("/error");

  }

  

  redirect("/logout");

}

  

export async function signInWithGoogle() {

  const supabase = await createClient();

  const { data, error } = await supabase.auth.signInWithOAuth({

    provider: "google",

    options: {

      queryParams: {

        access_type: "offline",

        prompt: "consent",

      },

    },

  });

  

  if (error) {

    console.log(error);

    redirect("/error");

  }

  

  redirect(data.url);

}
```

