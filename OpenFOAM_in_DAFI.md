[toc]

# OpenFOAM within DAFI environment

#### OpenFOAM compilation errors (April 29 th 2024) 

In `DAFI` ,  several tested`OpenFOAM` cases have been integrated with neural networks and the EnKF data assimilation method. I have encounted two `C++` errors when compiling these cases. 

##### `error: alphaField doesn't name a type `

##### `simpleFoam: symbol lookup error: /storage/home/vjc5126/OpenFOAM/vjc5126-2.3.x/platforms/linux64GccDPOpt/lib/libincompressibleRASModels.so: undefined symbol: _ZTIN4Foam14incompressible8RASModelE`

Both of these two errors were caused by incorrectly placing the `kOmegaQuadratic.C/H` files. They need to be located at a higher hierarchical level within the `incompressible` directory:

`~/OpenFOAM/user-v2312/src/TurbulenceModels/incompressible/turbulentTransportModels/RAS/kOmegaNNQuadratic`

while I used to put them under 

`/root/OpenFOAM/user-v2312/src/TurbulenceModels/turbulenceModels/RAS`

where you will need to include the ``kOmegaQuadratic.H` in the compile list in `/root/OpenFOAM/user-v2312/src/TurbulenceModels/incompressible/turbulentTransportModels/turbulentTransportModels.C` file.



If `kOmegaQuadratic.C/H` are forced to be placed at a lower hierarchical level under the `../incompressible/turbulentTransportModels`, you must `template` the new classes. This will result in errors like `error: alphaField doesn't name a type` . Such errors can be fixed by ensuring that the data types within the block under the **member functions** are consistent with the variables defined in the constructor, e.g.,



```c++
// * * * * * * * * * * * * * * * * Constructors  * * * * * * * * * * * * * * //
 80 
 81 kOmegaQuadratic::kOmegaQuadratic
 82 (
 83     const geometricOneField& alpha,
 84     const geometricOneField& rho,
 85     const volVectorField& U,
 86     const surfaceScalarField& alphaRhoPhi,
 87     const surfaceScalarField& phi,
 88     const transportModel& transport,
 89     const word& propertiesName,
 90     const word& type
 91 )
 92 :
8 // * * * * * * * * * * * * * * * Member Functions  * * * * * * * * * * * * * //
void kOmegaQuadratic::correct()
278 {
279     if (!turbulence_)
280     {
281         return;
282     }
283 
284     // Local references
285     const alphaField& alpha = alpha_; //if I chang alphaField& to geomtricOneField&
286     const rhoField& rho = rho_;
287     const surfaceScalarField& alphaRhoPhi = alphaRhoPhi_;
288     const volVectorField& U = U_;
289     const volScalarField& nut = nut_;
290     const volSymmTensorField& nonlinearStress = nonlinearStress_;
291     // fv::options& fvOptions(fv::options::New(this->mesh_));
292 
293     nonlinearEddyViscosity<incompressible::RASModel>::correct();
```



However, even it can be compiled successfully with these modifications applied. When you run your code under the `ensemble_learning` directory, you will get the error: `simpleFoam: symbol lookup error:` . This error still arises due to conflicts in dependencies caused by placing  `kOmegaQuadratic.C` under a lower hierarchical level.. 

The conclusion above is based on my findings and my current understanding so far. I will need to delve deeper into the codes under the `src/TurbulenceModels/incompressible` directory, which are written differently from the **templated turbulence models** under `/root/OpenFOAM/user-v2312/src/TurbulenceModels/turbulenceModels/RAS`. One of the key difference it uses three namespaces to define the scope of the code: `Foam, incompressible, RASModels`. 



**Note: the `kOmegaQuadratic.C/H`** **and the similar codes don't need to be modified at all, and just need to be put under the right place: ** **`~/OpenFOAM/user-v2312/src/TurbulenceModels/incompressible/turbulentTransportModels/RAS/`**
