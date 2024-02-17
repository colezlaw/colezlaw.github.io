+++
title = 'Possible Go encoding/json/v2'
date = 2024-02-17T11:48:44-05:00
draft = false

[cover]
    alt = "a computer screen with code behind a pair of glasses"
    caption = "Will `github.com/go-json-experiment` become the new `encoding/json`?"
    image = "/code.jpg"
    relative = false
+++

{{< youtube mBinOZ4KW7k >}}

While it's not perfect, I love the `encoding/json` package in Go's standard library.
However, as you can see from the video above, it does have its drawbacks. And while
none of the performance issues or bugs have been an issue for me in the past, the work
done on this new version does simplify something I run into frequently, which is
dealing with custom encoding and decoding for standard types.

I deal with a lot of "other people's API's", so I don't get to make the rules, I just have
to follow them. I get a lot of API's where instead of using JSON booleans for boolean values,
they use `"Y"` or `"N"`. And instead of using a consistent format for dates, it'll use
times with odd formats or a mix of ISO 8601 dates (with no time) on some fields and a full
time on others.

With the current `encoding/json` in the standard library, both of these cases require me
to make a new type and implement [`json.Marshaler`](https://pkg.go.dev/encoding/json#Marshaler)
or [`json.Unmarshaler`](https://pkg.go.dev/encoding/json#Unmarshaler). The [experimental replacement](https://pkg.go.dev/github.com/go-json-experiment/json) has a couple of features that really simplify this.

## Dealing with Date formats

First, the easy one is dealing with date formats. Rather than having to make a new `JSONDate` type and implementing `Marshaler` and/or `Unmarshaler`, you can now just specify the format in the struct tag:

```go
type Pet struct {
    Name string `json:"name"`
    DOB time.Time `json:"dob,format:'2006-01-02'"`
}
```

## Dealing with pseudo-booleans

To deal with formatting booleans as `Y` or `N`, you can use the [MarshalFuncV2](https://pkg.go.dev/github.com/go-json-experiment/json#MarshalFuncV2) or [UnmarshalFuncV2](https://pkg.go.dev/github.com/go-json-experiment/json#UnarshalFuncV2) as necessary.

```go
boolEncoder := json.MarshalFuncV2(func(e *jsontext.Encoder, b bool, opts json.Options) error {
	if b {
		e.WriteToken(jsontext.String("Y"))
	} else {
		e.WriteToken(jsontext.String("N"))
	}

    return nil
})

if err := json.MarshalWrite(out, result, json.WithMarshalers(boolEncoder)); err != nil {
    ...
}
```
###### Encoding a boolean as Y/N

And decoding looks similar.

```go
boolDecoder := json.UnmarshalFuncV2(func(d *jsontext.Decoder, b *bool, opts json.Options) error {
	*b = false
	
    t, err := d.ReadToken()
    if err != nil {
		return err
	}

	if t.Kind() != '"' {
		return fmt.Errorf("expected string got %c", t.Kind())
	}

	if t.String() == "Y" || t.String() == "y" {
		*b = true
	}

	return nil
})

if err := json.UnmarshalRead(in, &result, json.WithUnmarshalers(boolDecoder)); err != nil {
	...
}
```

## Final Thoughts

These aren't the only great things about the working experiment. It's not even a formal proposal just yet, but I look forward to the conversation getting started. One disappointing thing about the video is that he says if the proposal doesn't get off the ground, then this project gets abandoned. I certainly hope that's not the case - even if it's not chosen as a replacement for `encoding/json`, I think there's enough value here to keep the project going.
