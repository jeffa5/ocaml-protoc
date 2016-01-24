# ocaml-protoc

A [protobuf](https://developers.google.com/protocol-buffers/) compiler for OCaml. 

### Introduction 

`ocaml-protoc` compiles [protobuf message files](https://developers.google.com/protocol-buffers/docs/proto) into 
[OCaml modules](http://caml.inria.fr/pub/docs/manual-ocaml/moduleexamples.html). Each message/enum/oneof protobuf type 
will have a corresponding OCaml type along with the following functions:
* `encode_<type>` : encode the generated type to `bytes` following protobuf specification
* `decode_<type>` : decode the generated type from `bytes` following protobuf specification
* `default_<type>` : default value honoring [protobuf default attributes](https://developers.google.com/protocol-buffers/docs/proto#optional) or [protobuf version 3 default rules](https://developers.google.com/protocol-buffers/docs/proto3#default) 
* `pp_<type>` : pretty print of the OCaml type

### A simple example

Let's take a similar example as the [google one](https://developers.google.com/protocol-buffers/docs/overview#how-do-they-work):

```Javascript
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
  repeated string phone = 4;
}
```
The following OCaml code will get generated after running `ocaml-protoc -ml_out ./ example.proto`
```OCaml
type person = {
  mutable name : string;
  mutable id : int;
  mutable email : string option;
  mutable phone : string list;
}

val decode_person : Pbrt.Decoder.t -> person
(** [decode_person decoder] decodes a [person] value from [decoder] *)

val encode_person : person -> Pbrt.Encoder.t -> unit
(** [encode_person v encoder] encodes [v] with the given [encoder] *)

val pp_person : Format.formatter -> person -> unit 
(** [pp_person v] formats v] *)

val default_person : unit -> person
(** [default_person ()] is the default value for type [person] *)
```

You can then use this OCaml module in your application to populate, serialize, and retrieve `person` protocol buffer messages.
For example:

```OCaml
let () =

  (* Create OCaml value of generated type *) 
  let person = Example_pb.({ 
    name = "John Doe"; 
    id = 1234;
    email = Some "jdoe@example.com"; 
    phone = ["123-456-7890"];
  }) in 
  
  (* Create a Protobuf encoder and encode value *)
  let encoder = Pbrt.Encoder.create () in 
  Example_pb.encode_person person encoder; 

  (* Output the protobuf message to a file *) 
  let oc = open_out "myfile" in 
  output_bytes oc (Pbrt.Encoder.to_bytes encoder);
  close_out oc
```

Then later on you can read your message back in:
```OCaml
let () = 
  (* Read bytes from the file *) 
  let bytes = 
    let ic = open_in "myfile" in 
    let len = in_channel_length ic in 
    let bytes = Bytes.create len in 
    really_input ic bytes 0 len; 
    close_in ic; 
    bytes 
  in 
  
  (* Decode the person and Pretty-print it *)
  let person = Test14_pb.decode_person (Pbrt.Decoder.of_bytes bytes) in 
  Format.fprintf Format.std_formatter "%a" Test14_pb.pp_person person
```