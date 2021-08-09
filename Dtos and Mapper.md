# AlkemyNotesCsharp

# DTOs y Mapping

Creamos nuestro mapper, una clase a la que llamaremos "EntityMapper". Allí irán los métodos que pasen de una clase (model) a DTO. O viceversa.

1) Creamos el DTO en cuestión, una clase nueva.
  
  
```
  public class CategoryCreateDto
    {
        [Required]
        [DataType(DataType.Text)]
        public string Name { get; set; }
        [Required]
        [DataType(DataType.Text)]
        public string Description { get; set; }
        public IFormFile Image { get; set; }
    }
```

2) Creamos el mapper, con el método que controla este Dto:

```
  public class EntityMapper
    {
      public CategoryModel FromCategoryCreateDtoToCategory(CategoryCreateDto categoryCreateDto)
        {
            if (categoryCreateDto == null)
            {
                return null;
            }
            return new CategoryModel
            {
                Description = categoryCreateDto.Description,
                Image = "category_" + categoryCreateDto.Name,
                Name = categoryCreateDto.Name,
            };
        }
     }
```

3) Creamos en el servicio (no olvidar en la interfaz el encabezado del método) el método que lo utiliza, instanciando EntityMapper para acceder a él, y UnitOfWork para acceder al repositorio de ese modelo, de no usar el patrón UoW, instanciar el repositorio particular.

```
  private readonly IUnitOfWork _unitOfWork;

        public CategoryService(IUnitOfWork unitOfWork)
        {
            _unitOfWork = unitOfWork;
        }
        
  public async Task<CategoryModel> Post(CategoryCreateDto categoryCreateDto)
        {
            var mapper = new EntityMapper();
            var category = mapper.FromCategoryCreateDtoToCategory(categoryCreateDto);

            await _unitOfWork.CategoryRepository.Insert(category);
            await _unitOfWork.SaveChangesAsync();

            return category;
        }
```
4) En el controlador, incluímos el endpoint solicitado, instanciamos el servicio para poder consumirlo. Si queremos agregar un Tag Helper (como [FromForm]) este es el lugar, no otro. Vigilar que haya una respuesta correcta, por ejemplo, si creamos un objeto nuevo, debe devolver Created() Status Code 201. 

```
  private readonly ICategoryService _iCategoryService;

        public CategoryController(ICategoryService iCategoryService)
        {
            _iCategoryService = iCategoryService;
        }
        
  [HttpPost]

        public async Task<IActionResult> Post([FromForm] CategoryCreateDto categoryCreateDto)
        {
            if (!ModelState.IsValid)
                return BadRequest();
            try
            {
                var response = await _iCategoryService.Post(categoryCreateDto);

                return CreatedAtAction("POST", response);
            }
            catch (Exception e)
            {
                return BadRequest(e);
            }

        }
```
